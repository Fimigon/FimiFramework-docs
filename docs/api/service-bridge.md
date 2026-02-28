# Service Bridge

When a controller calls `Framework:GetService("ServiceName")` on the client, it receives a **Service Bridge** — a proxy object that communicates with the server through remotes.

The bridge is automatically constructed from the remotes the server created during initialization.

---

## Getting a Bridge

```lua
local service = Framework:GetService("PowerService")
```

The bridge is cached after first creation. Subsequent calls to `GetService` with the same name return the same bridge.

!!! danger "Timeout"
    If the server hasn't created the remotes for this service within 10 seconds, the framework throws an error. Ensure the server calls `InitServer` before clients connect.

---

## Methods

### `Service:MethodName(...: any) → ...any`

Calls a `Client` method defined on the service. This invokes the corresponding `RemoteFunction` and **yields** until the server responds.

```lua
local power = service:GetPower()
local data, err = service:GetPlayerData()
```

Each function defined in the service's `Client` table becomes a callable method on the bridge. The `player` argument is automatically passed by the remote system — you do not need to include it.

!!! note "Error handling"
    If the server method errors, the framework catches it and returns `nil, "Internal server error"`. If middleware blocks the request, it returns `nil, "Blocked by middleware"` (or a custom message).

---

### `Service:Fire(signalName: string, ...: any)`

Fires a `ClientSignal` to the server. This is a one-way fire-and-forget call — it does not return a value.

```lua
service:Fire("StartCharging")
service:Fire("RequestCoin", "coin_42")
```

This maps to `RemoteEvent:FireServer(...)` under the hood.

---

### `Service:On(signalName: string, callback: (...any) → ()) → RBXScriptConnection?`

Listens for a `Signal` fired from the server to this client.

```lua
service:On("PowerUpdated", function(power)
    print("Power:", power)
end)

service:On("CoinCollected", function(coinId, newScore)
    print("Got coin", coinId, "score:", newScore)
end)
```

Returns an `RBXScriptConnection` that can be disconnected.

---

### `Service:OnUnreliable(signalName: string, callback: (...any) → ()) → RBXScriptConnection?`

Listens for an `UnreliableSignal` fired from the server. These events may be dropped under network load.

```lua
service:OnUnreliable("PositionSync", function(position)
    -- Update position (may skip frames)
end)
```

---

### `Service:Destroy()`

Disconnects all connections made through `On()` and `OnUnreliable()` on this bridge.

```lua
service:Destroy()
```

---

## Summary Table

| Bridge Method | Corresponds To | Direction | Yields? |
|---|---|---|---|
| `Service:MethodName(...)` | `Client` method on service | Client → Server → Client | Yes |
| `Service:Fire(signal, ...)` | `ClientSignals` on service | Client → Server | No |
| `Service:On(signal, fn)` | `Signals` on service | Server → Client | No |
| `Service:OnUnreliable(signal, fn)` | `UnreliableSignals` on service | Server → Client | No |
| `Service:Destroy()` | Cleanup | — | No |