---
type: question
category: concurrency
difficulty: senior
tags: [flow, callbackflow, channelflow, channel]
created: 2026-06-16
last_reviewed:
times_failed: 0
times_passed: 0
status: new
---

# callbackFlow / channelFlow / Channel — обёртки над callbacks и concurrent emit

## Краткий ответ (TL;DR)

**`callbackFlow`** — обёртка над callback API (LocationCallback, BroadcastReceiver). Создаёт Channel внутри, `awaitClose` для cleanup. **`channelFlow`** — позволяет emit из нескольких корутин/dispatcher'ов (обычный `flow { }` строго sequential). **`Channel`** — низкоуровневая SPSC/MPMC очередь между корутинами, аналог Java `BlockingQueue`. `Channel.receiveAsFlow()` — превращает в hot Flow.

## Развёрнутый ответ

### callbackFlow

```kotlin
fun locationUpdates(client: FusedLocationProviderClient): Flow<Location> = callbackFlow {
    val request = LocationRequest.create().setInterval(1000)
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }
    client.requestLocationUpdates(request, callback, Looper.getMainLooper())
    awaitClose { client.removeLocationUpdates(callback) }
}
```

- `trySend` — non-suspending, возвращает `ChannelResult` (success/failed).
- `send` — suspending.
- `awaitClose { }` — **обязательно**. Вызывается при отписке collector'а (или cancel scope).

### channelFlow

Обычный `flow { }` нельзя emit из разных корутин:

```kotlin
flow {
    coroutineScope {
        launch { emit(1) }   // ❌ IllegalStateException — wrong context
    }
}
```

`channelFlow` решает:

```kotlin
fun merge(s1: Flow<Int>, s2: Flow<Int>): Flow<Int> = channelFlow {
    launch { s1.collect { send(it) } }
    launch { s2.collect { send(it) } }
}
```

- Создаёт Channel внутри.
- `send` — suspending, потокобезопасный.
- `trySend` — non-suspending.

### Когда channelFlow vs flow

- `flow { }` — sequential, без параллелизма. Дешевле (нет channel-overhead).
- `channelFlow` — параллельные источники, emit из launch'ей.

### Channel

```kotlin
val channel = Channel<Int>(capacity = Channel.BUFFERED)

launch {
    repeat(5) { channel.send(it) }
    channel.close()
}

launch {
    for (x in channel) println(x)
    // или: channel.consumeEach { println(it) }
}
```

### Channel capacity

- `Channel.RENDEZVOUS` (0) — send ждёт receive (по умолчанию).
- `Channel.UNLIMITED` — не блокирует send (memory risk).
- `Channel.CONFLATED` — только последнее значение.
- `Channel.BUFFERED` — default 64.
- N — фиксированный буфер.

### onBufferOverflow

```kotlin
Channel<Int>(
    capacity = 10,
    onBufferOverflow = BufferOverflow.DROP_OLDEST   // SUSPEND / DROP_LATEST / DROP_OLDEST
)
```

### Channel vs SharedFlow для events

- **Channel** = one-to-one (один subscriber получает каждое сообщение). Идеально для navigation events VM → UI (один collector).
- **SharedFlow** = one-to-many (broadcast). Подходит для UI-wide events.

```kotlin
// VM:
private val _navigation = Channel<NavCommand>()
val navigation = _navigation.receiveAsFlow()

fun goToProfile() = viewModelScope.launch {
    _navigation.send(NavCommand.ToProfile)
}
```

### produce (deprecated в пользу channelFlow)

```kotlin
val producer = scope.produce {
    repeat(5) { send(it) }
}
```

Современная замена: `channelFlow` + `launchIn`.

### Conversion

```kotlin
channel.receiveAsFlow()      // hot Flow, каждое значение одному subscriber'у
channel.consumeAsFlow()      // hot Flow, после первого collector channel закрывается
```

### Pitfalls callbackFlow

```kotlin
// ❌ забыл awaitClose
callbackFlow {
    val cb = registerCallback { trySend(it) }
    // нет awaitClose → leak listener
}

// ❌ send instead of trySend в синхронном callback
callbackFlow {
    val cb = SyncCallback { value ->
        send(value)   // suspending в синхронном callback! не скомпилируется
    }
}
```

### offer (deprecated)

`offer` → `trySend` (новый API возвращает `ChannelResult` вместо Boolean).

### BroadcastReceiver через callbackFlow

```kotlin
fun networkChanges(context: Context): Flow<Boolean> = callbackFlow {
    val receiver = object : BroadcastReceiver() {
        override fun onReceive(ctx: Context, intent: Intent) {
            trySend(isConnected(ctx))
        }
    }
    val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
    context.registerReceiver(receiver, filter)
    awaitClose { context.unregisterReceiver(receiver) }
}
```

### Sensor через callbackFlow

```kotlin
fun accelerometer(manager: SensorManager): Flow<SensorEvent> = callbackFlow {
    val sensor = manager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
    val listener = object : SensorEventListener {
        override fun onSensorChanged(e: SensorEvent) { trySend(e) }
        override fun onAccuracyChanged(s: Sensor, a: Int) {}
    }
    manager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_UI)
    awaitClose { manager.unregisterListener(listener) }
}
```

### Capacity hint для callbackFlow

```kotlin
callbackFlow {
    // ...
}.buffer(Channel.UNLIMITED)
```

Если callback может прилетать быстрее, чем consumer обрабатывает — буферизация.

## Подводные камни

- **`awaitClose` отсутствует** → leak callback registration.
- **`flow { emit() }` из launch** → IllegalStateException, нужен `channelFlow`.
- **`trySend` returns failed** игнорируется → потеря событий. Проверять или использовать `send` с UNLIMITED.
- **Channel без close** → consumer повисает на receive.
- **Channel для broadcast** — только один consumer получает каждое сообщение, остальные ничего.
- **`consumeAsFlow` повторно** → exception (channel уже закрыт первым collector).

## Связанные темы

- [[q-09-cold-vs-hot-flow]]
- [[q-10-sharedflow-stateflow-livedata]]
- [[../04-Architecture/q-05-side-effects]]

## Follow-up

- Зачем `callbackFlow` нужен и что делает `awaitClose`?
- Чем `channelFlow` отличается от `flow { }`?
- Когда Channel предпочтительнее SharedFlow?
