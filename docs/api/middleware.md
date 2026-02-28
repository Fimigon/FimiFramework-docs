# Middleware

Middleware lets you intercept all incoming client requests before they reach your service `Client` methods. Use it for rate limiting, authentication, input validation, or logging.

---

## Adding Middleware

Call `Framework:AddMiddleware()` **before** `Framework:InitServer()`.

```lua
local Framework = require(ReplicatedStorage.Shared.Framework)

Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    print(player.Name, "called", serviceName .. "." .. methodName)
    return true
end)

Framework:InitServer(ServicesFolder)
Framework:StartServer()
```

!!! warning
    Middleware must be registered before initialization. Adding middleware after `InitServer` will throw an error.

---

## Callback Signature

```lua
function(player: Player, serviceName: string, methodName: string, ...any) → (boolean, string?)
```

| Parameter | Description |
|---|---|
| `player` | The player who made the request |
| `serviceName` | Name of the service being called |
| `methodName` | Name of the `Client` method being invoked |
| `...` | Any arguments the client passed |

**Return values:**

- Return `true` to **allow** the request to proceed.
- Return `false` to **block** the request. The client receives `nil, "Blocked by middleware"`.
- Return `false, "Custom message"` to block with a custom error message.

---

## Execution Order

If multiple middleware functions are registered, they run **in order**. If any middleware returns `false`, the request is blocked immediately and subsequent middleware functions are skipped.

```lua
-- Middleware 1: Logging (runs first)
Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    print(`[LOG] {player.Name} → {serviceName}.{methodName}`)
    return true
end)

-- Middleware 2: Rate limiting (runs second)
Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    if isRateLimited(player) then
        return false, "Too many requests"
    end
    return true
end)

-- Middleware 3: Auth check (runs third, only if middleware 2 passed)
Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    if not isAuthenticated(player) then
        return false, "Not authenticated"
    end
    return true
end)
```

---

## Error Handling

If a middleware function itself errors (throws an exception), the framework catches it, prints a warning, and returns `nil, "Internal server error"` to the client. The request is **not** forwarded to the service method.

---

## What Middleware Applies To

Middleware only runs on **`Client` method** invocations (i.e., `RemoteFunction` calls). It does **not** intercept:

- `ClientSignals` (client → server `RemoteEvent` fires)
- `Signals` or `UnreliableSignals` (server → client)
- Internal `Events` or `Functions`

---

## Examples

### Rate Limiter

```lua
local requestCounts = {}
local LIMIT = 20 -- requests per second

game.Players.PlayerRemoving:Connect(function(player)
    requestCounts[player] = nil
end)

task.spawn(function()
    while true do
        task.wait(1)
        for player in requestCounts do
            requestCounts[player] = 0
        end
    end
end)

Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    requestCounts[player] = (requestCounts[player] or 0) + 1
    if requestCounts[player] > LIMIT then
        return false, "Rate limited"
    end
    return true
end)
```

### Input Validation

```lua
Framework:AddMiddleware(function(player, serviceName, methodName, ...)
    local args = { ... }
    for _, arg in args do
        if typeof(arg) == "Instance" then
            return false, "Instances not allowed as arguments"
        end
    end
    return true
end)
```