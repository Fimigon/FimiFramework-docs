# Framework

The `Framework` module is the core of FimiFramework. It handles initialization, lifecycle management, and provides access to services and controllers.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Framework = require(ReplicatedStorage.Shared.Framework)
```

!!! note "Not newable"
    `Framework` is a singleton. You do not create instances of it — you require it and call methods directly.

---

## Server Methods

### `Framework:InitServer(servicesFolder: Folder)`

Initializes the framework on the server. This must be called before `StartServer`.

- Scans `servicesFolder` recursively for all `ModuleScript` instances.
- Requires each module and registers it as a service.
- Creates all `RemoteFunction`, `RemoteEvent`, and `UnreliableRemoteEvent` instances based on service declarations.
- Creates internal signals and functions for each service.
- Calls `Init()` on every service.

```lua
local ServicesFolder = ServerScriptService:WaitForChild("Services")
Framework:InitServer(ServicesFolder)
```

!!! warning "Init must not yield"
    If a service's `Init()` method yields (e.g., calls `task.wait()`), the framework will warn and block until it completes. Move any yielding logic to `Start()`.

---

### `Framework:StartServer()`

Starts all services by calling `Start()` on each one in a separate thread via `task.spawn`.

```lua
Framework:StartServer()
```

Must be called after `InitServer`. Each service's `Start()` runs concurrently and can yield freely.

---

### `Framework:AddMiddleware(middlewareFn)`

Registers a middleware function that intercepts all incoming client requests. See [Middleware](middleware.md) for full details.

```lua
Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    return true -- Allow the request
end)
```

!!! warning
    Middleware must be added **before** calling `InitServer`.

---

## Client Methods

### `Framework:InitClient(controllersFolder: Folder)`

Initializes the framework on the client.

- Waits for the `FrameworkRemotes` folder in `ReplicatedStorage` (up to 10 seconds).
- Scans `controllersFolder` recursively for all `ModuleScript` instances.
- Requires each module and registers it as a controller.
- Creates internal signals and functions for each controller.
- Calls `Init()` on every controller.

```lua
local ControllersFolder = Players.LocalPlayer.PlayerScripts:WaitForChild("Controllers")
Framework:InitClient(ControllersFolder)
```

---

### `Framework:StartClient()`

Starts all controllers by calling `Start()` on each one in a separate thread via `task.spawn`.

```lua
Framework:StartClient()
```

---

## Shared Methods

### `Framework:GetService(serviceName: string) → Service | ServiceBridge`

Retrieves a service by name.

**On the server:** Returns the actual service table, giving you full access to all methods, properties, and internal events.

```lua
-- Server-side
local PowerService = Framework:GetService("PowerService")
PowerService:On("PowerChanged", function(player, power)
    print(player.Name, power)
end)
```

**On the client:** Returns a [ServiceBridge](service-bridge.md) — a lightweight proxy that communicates with the server through remotes.

```lua
-- Client-side
local PowerService = Framework:GetService("PowerService")
local power = PowerService:GetPower() -- Invokes RemoteFunction
```

!!! danger "Timeout"
    On the client, if the service's remote folder isn't found within 10 seconds, the framework throws an assertion error. Make sure the server is initialized before clients connect.

---

### `Framework:GetController(controllerName: string) → Controller`

Retrieves a controller by name. **Client-only.**

```lua
local PowerController = Framework:GetController("PowerController")
PowerController:On("LocalPowerChanged", function(power)
    print("Power:", power)
end)
```

!!! danger
    Calling `GetController` on the server will throw an assertion error.

---

## Lifecycle Order

The framework follows a strict initialization order:

```
1. InitServer / InitClient
   ├── Load all modules
   ├── Create remotes / signals
   └── Call Init() on each module (must not yield)

2. StartServer / StartClient
   └── Call Start() on each module (can yield, runs in task.spawn)
```

Because `Init()` runs synchronously and before `Start()`, you can safely call `Framework:GetService()` and `Framework:GetController()` inside `Init()` to store references for use in `Start()`.