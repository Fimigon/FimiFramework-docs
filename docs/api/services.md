# Services

Services are **server-side** modules that manage game logic, player data, and expose functionality to clients through automatically generated remotes.

---

## Defining a Service

A service is a ModuleScript that returns a table. The framework recognizes several special fields in this table:

```lua
local MyService = {
    Name = "MyService",

    -- Networking declarations
    Client = {},                       -- Methods clients can call (RemoteFunctions)
    Signals = { "SignalName" },        -- Server → Client events (RemoteEvents)
    UnreliableSignals = { "Name" },    -- Server → Client lossy events (UnreliableRemoteEvents)
    ClientSignals = { "ClientSig" },   -- Client → Server events (RemoteEvents)

    -- Internal declarations (no networking)
    Events = { "EventName" },          -- Server-to-server events
    Functions = {},                     -- Server-to-server invocable functions
}

return MyService
```

---

## Declarations

### `Client`

A table of functions that clients can invoke. Each function becomes a `RemoteFunction`.

```lua
MyService.Client = {}

function MyService.Client:GetScore(player: Player)
    return MyService._scores[player] or 0
end

function MyService.Client:SetNickname(player: Player, nickname: string)
    MyService._nicknames[player] = nickname
    return true
end
```

The first parameter is always the `Player` who made the request. The return value is sent back to the client.

!!! note
    If [middleware](middleware.md) is active, all `Client` methods pass through middleware before executing.

---

### `Signals`

An array of signal names. Each becomes a `RemoteEvent` that the **server fires** and the **client listens to**.

```lua
Signals = { "CoinCollected", "LevelUp" }
```

Fire these using `self:FireClient()` or `self:FireAllClients()`.

---

### `UnreliableSignals`

An array of signal names. Each becomes an `UnreliableRemoteEvent` — useful for frequent, non-critical data like position updates.

```lua
UnreliableSignals = { "PositionSync" }
```

Fire these using `self:FireClientUnreliable()` or `self:FireAllClientsUnreliable()`.

---

### `ClientSignals`

An array of signal names. Each becomes a `RemoteEvent` that the **client fires** and the **server listens to**.

```lua
ClientSignals = { "RequestCoin", "StartCharging" }
```

Listen for these using `self:OnClient()`.

---

### `Events`

An array of internal event names. These are local signals that never touch the network — used for service-to-service communication.

```lua
Events = { "ScoreChanged", "PlayerDied" }
```

Fire with `self:Fire()`, listen with `self:On()`.

---

### `Functions`

A table of invocable functions for service-to-service communication. These are synchronous and return values.

```lua
Functions = {
    GetPlayerLevel = function(self, player)
        return self._levels[player] or 1
    end,
}
```

Call with `self:Invoke("GetPlayerLevel", player)`.

---

## Methods

### Networking — Server to Client

#### `self:FireClient(player: Player, signalName: string, ...: any)`

Fires a `Signal` to a single client.

```lua
self:FireClient(player, "CoinCollected", coinId, newScore)
```

---

#### `self:FireAllClients(signalName: string, ...: any)`

Fires a `Signal` to all connected clients.

```lua
self:FireAllClients("LevelUp", playerName, newLevel)
```

---

#### `self:FireClientUnreliable(player: Player, signalName: string, ...: any)`

Fires an `UnreliableSignal` to a single client. Packets may be dropped under load.

```lua
self:FireClientUnreliable(player, "PositionSync", position)
```

---

#### `self:FireAllClientsUnreliable(signalName: string, ...: any)`

Fires an `UnreliableSignal` to all connected clients.

```lua
self:FireAllClientsUnreliable("PositionSync", position)
```

---

### Networking — Client to Server

#### `self:OnClient(signalName: string, callback: (player: Player, ...any) → ()) → RBXScriptConnection?`

Listens for a `ClientSignal` fired by a client.

```lua
self:OnClient("StartCharging", function(player)
    self:_startCharging(player)
end)
```

---

### Internal Events

#### `self:Fire(eventName: string, ...: any)`

Fires an internal `Event`. Other services listening with `On()` will receive it.

```lua
self:Fire("ScoreChanged", player, newScore)
```

---

#### `self:On(eventName: string, callback: (...any) → ()) → Connection?`

Listens for an internal `Event`.

```lua
-- From within the same service
self:On("ScoreChanged", function(player, score)
    print(player.Name, score)
end)

-- From another service
local CoinService = Framework:GetService("CoinService")
CoinService:On("ScoreChanged", function(player, score)
    print(player.Name, score)
end)
```

---

### Internal Functions

#### `self:Invoke(funcName: string, ...: any) → ...any`

Calls an internal `Function` and returns its result.

```lua
local level = self:Invoke("GetPlayerLevel", player)
```

---

## Lifecycle

### `Init()`

Called during `Framework:InitServer()`. Runs synchronously — **must not yield**.

Use `Init()` to set up references, connect events, and initialize state.

```lua
function MyService:Init()
    self._scores = {}

    self:OnClient("RequestCoin", function(player, coinId)
        -- Handle coin request
    end)
end
```

---

### `Start()`

Called during `Framework:StartServer()`. Runs in a `task.spawn` thread — **can yield freely**.

Use `Start()` for logic that depends on all services being initialized, or for loops and async work.

```lua
function MyService:Start()
    local OtherService = Framework:GetService("OtherService")
    OtherService:On("SomeEvent", function(...)
        -- React to other service
    end)

    while true do
        task.wait(1)
        self:_tick()
    end
end
```

---

## Complete Example

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)

local CoinService = {
    Name = "CoinService",
    Client = {},
    Signals = { "CoinCollected" },
    ClientSignals = { "RequestCoin" },
    Events = { "ScoreChanged" },
}

function CoinService.Client:GetScore(player: Player)
    return CoinService._scores[player] or 0
end

function CoinService:Init()
    self._scores = {}

    self:OnClient("RequestCoin", function(player, coinId)
        self._scores[player] = (self._scores[player] or 0) + 1
        self:FireClient(player, "CoinCollected", coinId, self._scores[player])
        self:Fire("ScoreChanged", player, self._scores[player])
    end)
end

function CoinService:Start()
    game.Players.PlayerRemoving:Connect(function(player)
        self._scores[player] = nil
    end)
end

return CoinService
```