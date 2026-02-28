# Inter-Service Communication

Services can communicate with each other using **internal Events** and **Functions**. These never touch the network — they're local signals that run entirely on the server.

---

## Pattern: Listening to Another Service's Events

Use `Framework:GetService()` on the server to get a direct reference to another service, then call `:On()` to listen for its internal events.

### StatsService listens to PowerService

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)

local StatsService = {
    Name = "StatsService",
    Client = {},
    Signals = {},
    ClientSignals = {},
    Events = {},
}

function StatsService:Init()
    -- Nothing to set up
end

function StatsService:Start()
    local PowerService = Framework:GetService("PowerService")

    -- React to PowerService's internal "PowerChanged" event
    PowerService:On("PowerChanged", function(player, power)
        print("[StatsService]", player.Name, "power is now", power)
        -- Could update leaderboard, save stats, etc.
    end)
end

return StatsService
```

!!! note "Why `Start()` and not `Init()`?"
    You should listen to other services in `Start()` because all services are guaranteed to be fully initialized by that point. While `GetService()` works in `Init()`, the other service's events might not be set up yet if it hasn't had its `Init()` called.

---

## Pattern: Invoking Another Service's Functions

If you need to **get data** from another service (not just react to events), use `Functions` and `:Invoke()`.

### DataService provides player data, CoinService reads it

```lua
-- DataService
local DataService = {
    Name = "DataService",
    Events = {},
    Functions = {
        GetPlayerData = function(self, player)
            return self._data[player] or {}
        end,
    },
    _data = {},
}

function DataService:Init()
    -- Load data when players join
end

return DataService
```

```lua
-- CoinService uses DataService
local CoinService = {
    Name = "CoinService",
    Events = {},
}

function CoinService:Start()
    local DataService = Framework:GetService("DataService")

    game.Players.PlayerAdded:Connect(function(player)
        local data = DataService:Invoke("GetPlayerData", player)
        print("Player coins:", data.coins or 0)
    end)
end

return CoinService
```

---

## When to Use Events vs Functions

| Use Case | Mechanism | Example |
|---|---|---|
| Notify other services that something happened | `Events` + `Fire` / `On` | "A player scored a point" |
| Request data from another service | `Functions` + `Invoke` | "What is this player's level?" |

Events are fire-and-forget — the sender doesn't wait for a response. Functions are synchronous calls that return a value.