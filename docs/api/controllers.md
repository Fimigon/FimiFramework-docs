# Controllers

Controllers are **client-side** modules that handle user input, UI, and communication with server services through service bridges.

---

## Defining a Controller

A controller is a ModuleScript that returns a table:

```lua
local MyController = {
    Name = "MyController",

    -- Internal declarations (no networking)
    Events = { "EventName" },      -- Controller-to-controller events
    Functions = {},                 -- Controller-to-controller invocable functions
}

return MyController
```

!!! note
    Controllers do not have `Client`, `Signals`, `ClientSignals`, or `UnreliableSignals` — those are service-only. Controllers communicate with the server through [Service Bridges](service-bridge.md).

---

## Declarations

### `Events`

An array of internal event names for controller-to-controller communication.

```lua
Events = { "LocalPowerChanged", "UIUpdated" }
```

Fire with `self:Fire()`, listen with `self:On()` from any controller.

---

### `Functions`

A table of invocable functions for controller-to-controller communication.

```lua
Functions = {
    GetCurrentPower = function(self)
        return self._power
    end,
}
```

Call with `self:Invoke("GetCurrentPower")` from any controller.

---

## Methods

### Internal Events

#### `self:Fire(eventName: string, ...: any)`

Fires an internal `Event`. Other controllers listening with `On()` will receive it.

```lua
self:Fire("LocalPowerChanged", newPower)
```

---

#### `self:On(eventName: string, callback: (...any) → ()) → Connection?`

Listens for an internal `Event`.

```lua
-- From within the same controller
self:On("LocalPowerChanged", function(power)
    print("Power:", power)
end)

-- From another controller
local PowerController = Framework:GetController("PowerController")
PowerController:On("LocalPowerChanged", function(power)
    print("Power:", power)
end)
```

---

### Internal Functions

#### `self:Invoke(funcName: string, ...: any) → ...any`

Calls an internal `Function` and returns its result.

```lua
local power = self:Invoke("GetCurrentPower")
```

---

## Accessing Server Services

Controllers access server services through `Framework:GetService()`, which returns a [ServiceBridge](service-bridge.md).

```lua
function MyController:Init()
    self._coinService = Framework:GetService("CoinService")
end

function MyController:Start()
    -- Call a server method (yields)
    local score = self._coinService:GetScore()

    -- Listen to server signals
    self._coinService:On("CoinCollected", function(coinId, newScore)
        print("Collected!", coinId, newScore)
    end)

    -- Fire a client signal to the server
    self._coinService:Fire("RequestCoin", "coin_123")
end
```

---

## Accessing Other Controllers

Use `Framework:GetController()` to reference another controller and interact with its events or functions.

```lua
function EffectsController:Start()
    local PowerController = Framework:GetController("PowerController")

    PowerController:On("LocalPowerChanged", function(power)
        if power >= 50 then
            print("Power is high! Play particles")
        end
    end)
end
```

---

## Lifecycle

### `Init()`

Called during `Framework:InitClient()`. Runs synchronously — **must not yield**.

Use `Init()` to grab service references and set up initial state.

```lua
function MyController:Init()
    self._service = Framework:GetService("MyService")
    self._score = 0
end
```

---

### `Start()`

Called during `Framework:StartClient()`. Runs in a `task.spawn` thread — **can yield freely**.

Use `Start()` to connect to signals, fetch initial data, and set up input handling.

```lua
function MyController:Start()
    self._service:On("ScoreUpdated", function(score)
        self._score = score
    end)

    local initialScore = self._service:GetScore()
    self._score = initialScore
end
```

---

## Complete Example

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local Framework = require(ReplicatedStorage.Shared.Framework)

local PowerController = {
    Name = "PowerController",
    Events = { "LocalPowerChanged" },
    _power = 0,
}

function PowerController:Init()
    self._service = Framework:GetService("PowerService")
end

function PowerController:Start()
    self._service:On("PowerUpdated", function(power)
        self._power = power
        self:Fire("LocalPowerChanged", power)
    end)

    self._service:On("PlayerMaxedOut", function(playerName)
        print(playerName .. " reached MAX POWER!")
    end)

    UserInputService.InputBegan:Connect(function(input, processed)
        if processed then return end
        if input.KeyCode == Enum.KeyCode.E then
            self._service:Fire("StartCharging")
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.KeyCode == Enum.KeyCode.E then
            self._service:Fire("StopCharging")
        end
    end)

    self._power = self._service:GetPower()
end

function PowerController:GetPower(): number
    return self._power
end

return PowerController
```