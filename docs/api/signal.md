# Signal

The `Signal` module is a lightweight event system used internally by FimiFramework for `Events` and `Functions` declared on services and controllers.

!!! note
    You typically don't interact with the Signal module directly. The framework creates and manages signals for you when you declare `Events` on a service or controller. This page is provided for reference.

---

## Location

```lua
local Signal = require(ReplicatedStorage.Shared.Signal)
```

---

## Creating a Signal

```lua
local mySignal = Signal()
```

Returns a new signal object.

---

## Methods

### `Signal:Fire(...: any)`

Fires the signal, invoking all connected callbacks with the provided arguments.

```lua
mySignal:Fire("hello", 42)
```

---

### `Signal:Connect(callback: (...any) → ()) → Connection`

Connects a callback function to the signal. Returns a connection object.

```lua
local connection = mySignal:Connect(function(message, number)
    print(message, number)
end)
```

---

### `Connection:Disconnect()`

Disconnects a previously connected callback.

```lua
connection:Disconnect()
```

---

## How the Framework Uses Signal

When you declare `Events` on a service or controller:

```lua
local MyService = {
    Events = { "ScoreChanged" },
}
```

The framework internally creates `Signal()` instances for each event name. When you call `self:Fire("ScoreChanged", ...)` or `self:On("ScoreChanged", fn)`, you're interacting with these signals through the convenience methods the framework adds to your service/controller table.

These signals are purely local — they never create or use any `RemoteEvent` instances. For networking, the framework uses the `Signals`, `ClientSignals`, and `UnreliableSignals` declarations instead, which map to actual Roblox remote instances.