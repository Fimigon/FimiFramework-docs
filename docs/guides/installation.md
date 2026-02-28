# Installation

## Requirements

- Roblox Studio
- A basic understanding of Luau scripting

## Setup

### 1. Create the Shared folder

Inside `ReplicatedStorage`, create a folder called **Shared**. This folder will hold the framework module and the signal module.

```
ReplicatedStorage/
└── Shared/
    ├── Framework    (ModuleScript)
    └── Signal       (ModuleScript)
```

### 2. Add the Framework module

Create a **ModuleScript** named `Framework` inside `ReplicatedStorage.Shared` and paste the full FimiFramework source code into it.

### 3. Add the Signal module

Create a **ModuleScript** named `Signal` inside `ReplicatedStorage.Shared` and paste the Signal module source code into it. This is a dependency used internally by the framework for internal events.

### 4. Create the Services folder

Inside `ServerScriptService`, create a folder called **Services**. All of your service modules will go here.

```
ServerScriptService/
├── Services/
│   ├── MyService       (ModuleScript)
│   └── AnotherService  (ModuleScript)
└── Script              (Server Script — bootstrapper)
```

### 5. Create the Controllers folder

Inside `StarterPlayer > StarterPlayerScripts`, create a folder called **Controllers**. All of your controller modules will go here.

```
StarterPlayer/
└── StarterPlayerScripts/
    ├── Controllers/
    │   ├── MyController       (ModuleScript)
    │   └── AnotherController  (ModuleScript)
    └── Client                 (LocalScript — bootstrapper)
```

### 6. Create the bootstrap scripts

You need two scripts to initialize the framework — one on the server and one on the client.

=== "Server Script"

    Create a **Script** inside `ServerScriptService`:

    ```lua
    local ServerScriptService = game:GetService("ServerScriptService")
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local Framework = require(ReplicatedStorage.Shared.Framework)

    local ServicesFolder = ServerScriptService:WaitForChild("Services")

    Framework:InitServer(ServicesFolder)
    Framework:StartServer()

    print("[Server] Framework started")
    ```

=== "Client Script"

    Create a **LocalScript** inside `StarterPlayerScripts`:

    ```lua
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local Players = game:GetService("Players")
    local Framework = require(ReplicatedStorage.Shared.Framework)

    local ControllersFolder = Players.LocalPlayer.PlayerScripts:WaitForChild("Controllers")

    Framework:InitClient(ControllersFolder)
    Framework:StartClient()

    print("[Client] Framework started")
    ```

!!! warning "Boot order matters"
    The server must call `InitServer` and `StartServer` before any client connects. The client bootstrap waits for the `FrameworkRemotes` folder in `ReplicatedStorage`, but will time out after 10 seconds if the server hasn't initialized.

---

## Verification

After setting everything up, press **Play** in Roblox Studio. You should see:

```
[Server] Framework started
[Client] Framework started
```

in the output window. If you see errors about missing modules, double-check that `Framework` and `Signal` are both inside `ReplicatedStorage.Shared`.