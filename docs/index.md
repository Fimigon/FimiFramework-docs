# FimiFramework

**FimiFramework** is a Roblox Luau framework that provides a structured service/controller architecture with built-in networking, middleware support, and internal event systems.

---

## What is FimiFramework?

FimiFramework organizes your game code into two core concepts:

- **Services** run on the **server**. They manage game state, handle player data, and expose methods and signals to clients.
- **Controllers** run on the **client**. They handle user input, UI updates, and communicate with services through automatically generated networking bridges.

The framework handles all `RemoteFunction` and `RemoteEvent` creation for you — just declare what you need in a table, and FimiFramework wires everything up at runtime.

---

## Key Features

- **Automatic networking** — Declare `Client` methods, `Signals`, `ClientSignals`, and `UnreliableSignals` on services. FimiFramework creates all remotes automatically.
- **Service bridges** — Controllers access server services through bridges with a clean API. No manual remote setup required.
- **Middleware** — Intercept and validate all incoming client requests before they reach your service methods.
- **Internal events** — Services can communicate with other services, and controllers with other controllers, using internal `Events` and `Functions` that never touch the network.
- **Lifecycle hooks** — `Init()` and `Start()` give you clear phases for setup and runtime logic.

---

## Quick Example

=== "Server (Service)"

    ```lua
    local MyService = {
        Name = "MyService",
        Client = {},
        Signals = { "Notify" },
    }

    function MyService.Client:GetData(player: Player)
        return { coins = 100 }
    end

    function MyService:Start()
        self:FireAllClients("Notify", "Server is ready!")
    end

    return MyService
    ```

=== "Client (Controller)"

    ```lua
    local Framework = require(ReplicatedStorage.Shared.Framework)

    local MyController = {
        Name = "MyController",
    }

    function MyController:Start()
        local service = Framework:GetService("MyService")
        service:On("Notify", function(message)
            print(message)
        end)

        local data = service:GetData()
        print(data.coins) -- 100
    end

    return MyController
    ```

---

## Next Steps

- [Installation](guides/installation.md) — Set up FimiFramework in your project.
- [Project Structure](guides/project-structure.md) — Understand how to organize your files.
- [Quick Start](guides/quick-start.md) — Build your first service and controller.