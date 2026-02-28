# Power System Example

This example demonstrates a complete power charging system using FimiFramework. Players hold a key to charge power, the server tracks the state, and clients receive real-time updates.

---

## Overview

- The **client** detects when the player holds/releases the `E` key and fires signals to the server.
- The **server** tracks which players are charging, increments their power every 0.5 seconds, and fires updates back.
- When a player hits max power, the server notifies all clients and resets.

---

## PowerService (Server)

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Framework = require(ReplicatedStorage.Shared.Framework)

local PowerService = {
    Name = "PowerService",
    Client = {},
    Signals = {
        "PowerUpdated",
        "PlayerMaxedOut"
    },
    ClientSignals = {
        "StartCharging",
        "StopCharging"
    },
    Events = {
        "PowerChanged"
    },
    _playerPower = {},
    _chargingPlayers = {}
}

local MAX_POWER = 100
local CHARGE_RATE = 10

-- Client can request their current power
function PowerService.Client:GetPower(player: Player): number
    return PowerService._playerPower[player] or 0
end

function PowerService:Init()
    Players.PlayerAdded:Connect(function(player)
        self._playerPower[player] = 0
    end)

    Players.PlayerRemoving:Connect(function(player)
        self._playerPower[player] = nil
        self._chargingPlayers[player] = nil
    end)

    self:OnClient("StartCharging", function(player)
        self._chargingPlayers[player] = true
    end)

    self:OnClient("StopCharging", function(player)
        self._chargingPlayers[player] = nil
    end)
end

function PowerService:Start()
    -- Initialize power for any players already in the server
    for _, player in Players:GetPlayers() do
        self._playerPower[player] = 0
    end

    -- Main charge loop
    while true do
        task.wait(0.5)
        for player, _ in self._chargingPlayers do
            local current = self._playerPower[player] or 0
            local newPower = math.min(current + CHARGE_RATE, MAX_POWER)
            self._playerPower[player] = newPower

            -- Notify the charging player
            self:FireClient(player, "PowerUpdated", newPower)

            -- Notify other services
            self:Fire("PowerChanged", player, newPower)

            -- Check for max power
            if newPower >= MAX_POWER and current < MAX_POWER then
                self:FireAllClients("PlayerMaxedOut", player.Name)
                self._playerPower[player] = 0
                self._chargingPlayers[player] = nil
                self:FireClient(player, "PowerUpdated", 0)
            end
        end
    end
end

return PowerService
```

---

## PowerController (Client)

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
    -- Listen to server signals
    self._service:On("PowerUpdated", function(power)
        self._power = power
        self:Fire("LocalPowerChanged", power)
    end)

    self._service:On("PlayerMaxedOut", function(playerName)
        print(playerName .. " reached MAX POWER!")
    end)

    -- Input: hold E to charge
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

    -- Fetch initial power
    self._power = self._service:GetPower()
end

function PowerController:GetPower(): number
    return self._power
end

return PowerController
```

---

## Communication Flow

```
Client (hold E)
  │
  ├──Fire("StartCharging")──→  Server (PowerService)
  │                              │
  │                              ├── Tick loop: power += 10
  │                              │
  │  ◄──FireClient("PowerUpdated")──┤
  │                              │
  │                              ├──Fire("PowerChanged")──→ Other Services
  │                              │
  │  ◄──FireAllClients("PlayerMaxedOut")──┤  (when max reached)
  │
  ├──Fire("StopCharging")──→  Server
```