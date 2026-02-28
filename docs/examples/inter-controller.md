# Inter-Controller Communication

Controllers can communicate with each other using **internal Events** and **Functions**, just like services. These are local signals that run entirely on the client.

---

## Pattern: Reacting to Another Controller's Events

Use `Framework:GetController()` to reference another controller and listen to its events.

### EffectsController reacts to PowerController

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)

local EffectsController = {
    Name = "EffectsController",
    Events = {},
}

function EffectsController:Init()
    -- Nothing to set up
end

function EffectsController:Start()
    local PowerController = Framework:GetController("PowerController")

    PowerController:On("LocalPowerChanged", function(power)
        if power >= 50 then
            print("[Effects] Power is high! Play particles")
        elseif power > 0 then
            print("[Effects] Power is charging...")
        else
            print("[Effects] Power reset")
        end
    end)
end

return EffectsController
```

This pattern lets you keep your controllers focused on single responsibilities. `PowerController` manages power state, `EffectsController` manages visual effects, and they communicate through events without being tightly coupled.

---

## Pattern: Reading Another Controller's State

Use `Functions` and `:Invoke()` to read data from another controller, or simply call public methods on the controller reference.

### UIController reads PowerController's state

```lua
local UIController = {
    Name = "UIController",
    Events = {},
}

function UIController:Start()
    local PowerController = Framework:GetController("PowerController")

    -- Option 1: Call a regular method
    local power = PowerController:GetPower()
    print("Current power:", power)

    -- Option 2: Listen for updates
    PowerController:On("LocalPowerChanged", function(power)
        -- Update a UI element
        self:_updatePowerBar(power)
    end)
end

function UIController:_updatePowerBar(power: number)
    -- Update your UI here
end

return UIController
```

!!! note
    You can call any public method on a controller reference directly — you don't need to use `Invoke` for simple method calls. `Events` and `Functions`/`Invoke` are specifically for the framework's managed signal system.

---

## Communication Diagram

```
PowerController                     EffectsController
     │                                    │
     │  Fire("LocalPowerChanged", 50)     │
     ├────────────────────────────────────►│
     │                                    ├── Play particles
     │                                    │
     │                                UIController
     │                                    │
     │  Fire("LocalPowerChanged", 50)     │
     ├────────────────────────────────────►│
     │                                    ├── Update power bar
```

Multiple controllers can listen to the same event. The firing controller doesn't need to know who's listening.