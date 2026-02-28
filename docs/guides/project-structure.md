# Project Structure

FimiFramework expects a specific folder layout in your Roblox place. Here's the full recommended structure:

```
ReplicatedStorage/
└── Shared/
    ├── Framework        (ModuleScript)  -- The core framework module
    └── Signal           (ModuleScript)  -- Signal utility module

ServerScriptService/
├── Services/            (Folder)        -- All service modules go here
│   ├── PowerService     (ModuleScript)
│   ├── StatsService     (ModuleScript)
│   └── ...
└── Script               (Script)        -- Server bootstrap script

StarterPlayer/
└── StarterPlayerScripts/
    ├── Controllers/     (Folder)        -- All controller modules go here
    │   ├── PowerController    (ModuleScript)
    │   ├── EffectsController  (ModuleScript)
    │   └── ...
    └── Client           (LocalScript)   -- Client bootstrap script
```

---

## Folder Breakdown

### `ReplicatedStorage/Shared`

This folder holds modules that both the server and client need access to. The `Framework` module and `Signal` module live here.

!!! note
    You can also place other shared utilities, types, or constants in this folder. Both services and controllers can require anything inside `ReplicatedStorage`.

### `ServerScriptService/Services`

Every **ModuleScript** inside this folder (including nested subfolders) is automatically discovered and loaded when you call `Framework:InitServer(ServicesFolder)`.

Each module must return a service table. The framework assigns the module's name as the service `Name` if you don't set one explicitly.

### `StarterPlayerScripts/Controllers`

Every **ModuleScript** inside this folder (including nested subfolders) is automatically discovered and loaded when you call `Framework:InitClient(ControllersFolder)`.

Each module must return a controller table. The framework assigns the module's name as the controller `Name` if you don't set one explicitly.

---

## Subfolder Support

Both the `Services` and `Controllers` folders support nesting. The framework recursively scans all children, so you can organize by feature:

```
Services/
├── Combat/
│   ├── DamageService    (ModuleScript)
│   └── WeaponService    (ModuleScript)
├── Economy/
│   ├── CoinService      (ModuleScript)
│   └── ShopService      (ModuleScript)
└── PlayerDataService    (ModuleScript)
```

Only `ModuleScript` instances are loaded — `Folder` instances are traversed but ignored otherwise.

---

## Runtime Remotes

When the server initializes, FimiFramework creates a folder in `ReplicatedStorage` called `FrameworkRemotes`. This is generated automatically and should not be modified manually.

```
ReplicatedStorage/
└── FrameworkRemotes/     (auto-generated)
    ├── PowerService/
    │   ├── GetPower              (RemoteFunction)
    │   ├── Signals/
    │   │   ├── PowerUpdated      (RemoteEvent)
    │   │   └── PlayerMaxedOut    (RemoteEvent)
    │   └── ClientSignals/
    │       ├── StartCharging     (RemoteEvent)
    │       └── StopCharging      (RemoteEvent)
    └── ...
```

!!! warning
    Do not rename, move, or delete anything inside `FrameworkRemotes`. The client relies on this folder structure to build service bridges.