# Quick Start

This guide walks you through creating your first service and controller using FimiFramework.

---

## Step 1: Create a Service

Create a **ModuleScript** called `GreetService` inside `ServerScriptService/Services`:

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)

local GreetService = {
    Name = "GreetService",
    Client = {},
    Signals = { "Greeted" },
}

function GreetService.Client:GetGreeting(player: Player)
    return "Hello, " .. player.Name .. "!"
end

function GreetService:Init()
    -- Init runs first. Must not yield.
    print("[GreetService] Initialized")
end

function GreetService:Start()
    -- Start runs after all services have initialized. Can yield.
    game.Players.PlayerAdded:Connect(function(player)
        task.wait(2)
        self:FireClient(player, "Greeted", "Welcome to the game!")
    end)
end

return GreetService
```

This service:

- Exposes a `GetGreeting` method that clients can call.
- Declares a `Greeted` signal the server can fire to individual clients.
- Sends a welcome message 2 seconds after a player joins.

---

## Step 2: Create a Controller

Create a **ModuleScript** called `GreetController` inside `StarterPlayerScripts/Controllers`:

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)

local GreetController = {
    Name = "GreetController",
}

function GreetController:Init()
    self._greetService = Framework:GetService("GreetService")
end

function GreetController:Start()
    -- Listen for the server greeting signal
    self._greetService:On("Greeted", function(message)
        print("[Client] Server says:", message)
    end)

    -- Call the server method
    local greeting = self._greetService:GetGreeting()
    print("[Client]", greeting)
end

return GreetController
```

This controller:

- Gets a bridge to `GreetService` during `Init()`.
- Listens for the `Greeted` signal from the server.
- Calls `GetGreeting()` which invokes the server's `RemoteFunction` and returns the result.

---

## Step 3: Run

Press **Play** in Roblox Studio. You should see output like:

```
[Server] Framework started
[GreetService] Initialized
[Client] Framework started
[Client] Hello, Player1!
[Client] Server says: Welcome to the game!
```

---

## What Just Happened?

1. The server bootstrap called `Framework:InitServer()` which loaded `GreetService`, created its `RemoteFunction` (`GetGreeting`) and `RemoteEvent` (`Greeted`), then called `Init()` and `Start()`.
2. The client bootstrap called `Framework:InitClient()` which loaded `GreetController`, called `Init()` (which grabbed the service bridge), then called `Start()`.
3. The controller called `self._greetService:GetGreeting()` — this invoked the `RemoteFunction` on the server, which returned the greeting string.
4. When the server fired `self:FireClient(player, "Greeted", ...)`, the client's `On("Greeted", ...)` callback received the message.

---

## Next Steps

- Learn the full [Service API](../api/services.md) to see all declaration options.
- Learn the full [Controller API](../api/controllers.md) to see what controllers can do.
- Add [Middleware](../api/middleware.md) to validate incoming client requests.