# RbxConfig

A singleton wrapper around Roblox `ConfigService`. The server reads cloud config and syncs it to clients.

Use this module for cloud-hosted settings. For local Studio config (Configuration instances, Attributes, or tables), use [BetterConfig](../BetterConfig/README.md).

## Installation

Wally package name: `zac134/rbx-config`. Depends on `sleitnick/signal`.

```toml
[dependencies]
RbxConfig = "zac134/rbx-config@0.0.2"
```

For a manual install, copy `Modules/RbxConfig/` into ReplicatedStorage and also provide `sleitnick/signal@^2.0`. The module `require`s `script.Parent.signal`.

Before use, create keys in [Creator Dashboard](https://create.roblox.com/) under **Configure → Config Service**. Key names must match the defaults table passed to `InitServer` / `InitClient`. See the [ConfigService documentation](https://create.roblox.com/docs/cloud/open-cloud/usage-configuration).

## Quick Start

Pass the same defaults table on the server and the client. Putting it in a shared module is the simplest approach.

```lua
-- Shared defaults (e.g. ReplicatedStorage/shared/RbxConfigSetting)
return {
    max_players_per_team = 4,
}
```

```lua
-- Server
local RbxConfig = require(ReplicatedStorage.Modules.RbxConfig):InitServer(configSettings)
print(RbxConfig:GetValue("max_players_per_team"))

RbxConfig:GetValueChangedSignal("max_players_per_team"):Connect(function(newValue)
    print("changed:", newValue)
end)
```

```lua
-- Client
local RbxConfig = require(ReplicatedStorage.Modules.RbxConfig):InitClient(configSettings)
print(RbxConfig:GetValue("max_players_per_team"))
```

Longer examples:

- [`/Examples/shared/RbxConfigSetting.luau`](../../Examples/shared/RbxConfigSetting.luau)
- [`/Examples/server/RbxConfig.server.luau`](../../Examples/server/RbxConfig.server.luau)
- [`/Examples/client/RbxConfig.client.luau`](../../Examples/client/RbxConfig.client.luau)

## Behavior

- Singleton. Call `InitServer` / `InitClient` once each. A second call warns and is ignored.
- On server init, defaults are copied into `_values`, then `ConfigService:GetConfigAsync()` fetches a global snapshot once and overwrites the same keys. If that fetch fails, defaults remain.
- On `InitClient`, the client requests the server's `_values` via `RemoteFunction` (`RbxConfigRemoteFunction`). On failure it uses the defaults you passed.
- Later updates are delivered per key via `RemoteEvent` (`RbxConfigRemoteEvent`).

### Value priority

1. Test override (`SetTestingValue`; server-session only)
2. Snapshot-backed `_values`
3. Defaults passed to `InitServer` / `InitClient`

### Current limitations

**Lazy observation**  
ConfigService live updates are subscribed and pushed to clients only for keys that have had `GetValueChangedSignal` called. Keys read only with `GetValue` may stay at the initial snapshot. Changing the cloud value does not update those keys automatically. Call `GetValueChangedSignal` on a key if you need live updates.

**Per-player snapshots**  
`GetValueForPlayer` caches `ConfigService:GetConfigForPlayerAsync` on first use and drops the cache on `PlayerRemoving`. It does not refetch or `Refresh`, so cloud changes after join are not applied.

**Test overrides**  
`SetTestingValue` / `ClearTestingValue` last only for the server process. They are lost on restart. Setting an override immediately pushes to all clients (whether or not the key is being observed). Do not use this for production logic.

## API

### `InitServer(configSettings) → ServerConfigClass`

Initialize on the server. The return value includes `GetValue` / `GetValueChangedSignal` plus the server-only methods.

### `InitClient(configSettings) → RbxConfig`

Initialize on the client. The return value has `GetValue` and `GetValueChangedSignal` only. `configSettings` should use the same keys and defaults as the server.

### Shared (server and client)

| Method | Description |
|---|---|
| `GetValue(key)` | Returns the current value using the priority above. |
| `GetValueChangedSignal(key)` | Signal that fires when the value changes (`Connect` / `Once` / `Wait`). On the server, this call starts ConfigService observation for that key. On the client, it fires when a `RemoteEvent` arrives from the server. |

### Server only

| Method | Description |
|---|---|
| `GetValueForPlayer(key, player)` | Returns the player-targeted snapshot value. Test overrides win if set. On fetch failure, falls back to the global value or the default. |
| `SetTestingValue(key, value)` | Sets a test override, fires the signal, and pushes to all clients. |
| `ClearTestingValue(key)` | Clears the override, restores the snapshot (or default), and pushes to clients. |

## Comparison with BetterConfig

| | RbxConfig | BetterConfig |
|---|---|---|
| Source | Cloud (ConfigService) | Local (Configuration / Attribute / table) |
| Server | Required | Not required |
| Remote update without restart | Observed keys only (see limitation above) | No (local only) |
| Per-player values | First-call cached `GetValueForPlayer` | None |

Use RbxConfig for cloud A/B tests and feature flags. Use BetterConfig for local Studio config.

## License

MIT License
