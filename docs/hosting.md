# Hosting BeatTogether.UnifiedServer

BeatTogether.UnifiedServer is an all-in-one multiplayer private server for Beat Saber. It bundles the Master Server, Dedicated Server Instancing, and Status API into a single process running on .NET 6. It supports crossplay between PC and Quest for game versions 1.37.0 and above.

## Requirements

- .NET 6 SDK (for building from source) or .NET 6 Runtime (for pre-built binaries)
- The following ports accessible from the internet:

| Port range    | Protocol | Purpose                        |
| ------------- | -------- | ------------------------------ |
| 8989          | TCP      | Master server & status API     |
| 30000–40000   | UDP      | Dedicated server instances     |

The UDP range is determined by `BasePort` and `MaximumSlots` in the configuration (see below). The defaults are `BasePort: 30000` and `MaximumSlots: 10000`, so open `30000–40000` UDP unless you change those values.

---

## Running from a Pre-built Release

1. Download the latest release for your platform from the [Releases](../../releases) page.
2. Extract the archive.
3. Edit `appsettings.json` (see [Configuration](#configuration)).
4. Run the binary:
   ```sh
   ./BeatTogether.UnifiedServer        # Linux / macOS
   BeatTogether.UnifiedServer.exe      # Windows
   ```

---

## Building from Source

Clone the repository with submodules, then publish for your target platform:

```sh
git clone --recurse-submodules https://github.com/BeatTogether/BeatTogether.UnifiedServer
cd BeatTogether.UnifiedServer
```

### Publish profiles

The repository ships publish profiles for the following targets. Use `dotnet publish` with the appropriate runtime identifier:

| Profile       | RID            |
| ------------- | -------------- |
| linux-x64     | `linux-x64`    |
| linux-arm64   | `linux-arm64`  |
| osx-x64       | `osx-x64`      |
| osx-arm64     | `osx-arm64`    |
| win-x64       | `win-x64`      |

Example — build a self-contained single-file binary for Linux x64:

```sh
dotnet publish BeatTogether.UnifiedServer.sln \
  -c Release \
  -r linux-x64 \
  -p:PublishSingleFile=true \
  -p:PublishTrimmed=false \
  -o ./out
```

The output binary is placed in `./out/BeatTogether.UnifiedServer`.

---

## Running with Docker

A `Dockerfile` is included in the repository root. It builds for `linux-x64`.

```sh
# Build the image
docker build -t beattogether-unified .

# Run the container
docker run -d \
  --name beattogether-unified \
  -p 8989:8989 \
  -p 30000-40000:30000-40000/udp \
  -v $(pwd)/appsettings.json:/app/appsettings.json \
  beattogether-unified
```

Mount your customised `appsettings.json` into `/app/appsettings.json` so that configuration survives container rebuilds.

---

## Configuration

All configuration for UnifiedServer lives in `appsettings.json` next to the binary. The file is split into four top-level sections.

### Example UnifiedServer Config

```json
{
  "Urls": "http://0.0.0.0:8989",

  "Serilog": {
    "File": {
      "Path": "logs/BeatTogether.UnifiedServer-{Date}.log"
    },
    "MinimumLevel": {
      "Default": "Information",
      "Overrides": {
        "Microsoft": "Warning"
      }
    }
  },

  "Status": {
    "MinimumAppVersion": "1.37.0",
    "ServerDisplayName": "My BeatTogether Server",
    "ServerDescription": "An awesome custom server",
    "ServerImageUrl": "",
    "UseSsl": false,
    "MaxPlayers": 30,
    "ServerSupportsPPModifiers": true,
    "ServerSupportsPPDifficulties": true,
    "ServerSupportsPPMaps": false,
    "MaintenanceStartTime": 0,
    "MaintenanceEndTime": 0,
    "LocalizedMessages": [
      {
        "Language": 0,
        "Message": "Server is going down for maintenance."
      }
    ],
    "RequiredMods": [
      { "id": "MultiplayerCore", "version": "1.2.0" },
      { "id": "BeatTogether",    "version": "2.0.1"  }
    ]
  },

  "Quickplay": {
    "PredefinedPacks": [
      { "order": 0, "packId": "ALL_LEVEL_PACKS" },
      { "order": 1, "packId": "BUILT_IN_LEVEL_PACKS" }
    ],
    "LocalizedCustomPacks": [
      {
        "serializedName": "customlevels",
        "order": 2,
        "localizedNames": [
          { "language": "English", "packName": "Custom" }
        ],
        "packIds": ["custom_levelpack_CustomLevels"]
      }
    ]
  },

  "ServerConfiguration": {
    "HostEndpoint": "YOUR_PUBLIC_IP", // <- Replace this!
    "UDPBindAddress": "0.0.0.0",
    "BasePort": 30000,
    "MaximumSlots": 100,
    "SessionTimeToLive": 180,
    "AuthenticateClients": true,
    "AuthBypassAllowed": false,
    "AuthedClients": ["OculusQuest", "Steam"],
    "VersionRanges": [
      { "MinVersion": "1.37.0", "MaxVersion": "2.0.0" }
    ]
  }
}
```

### `Urls`

The HTTP address the server listens on. `0.0.0.0` binds all interfaces. Change the port here if `8989` is already in use.

### `Serilog`

Controls logging behaviour.

| Key | Description |
| --- | ----------- |
| `File.Path` | Path template for rolling log files. `{Date}` is replaced with the current date. |
| `MinimumLevel.Default` | Minimum log level (`Verbose`, `Debug`, `Information`, `Warning`, `Error`). |
| `MinimumLevel.Overrides` | Per-namespace overrides. |

### `Status`

Returned by the `/status` endpoint and consumed by the BeatTogether mod to display server info on the server browser.

| Key | Description |
| --- | ----------- |
| `MinimumAppVersion` | Minimum Beat Saber version allowed to connect. |
| `ServerDisplayName` | Name shown in the in-game server browser. |
| `ServerDescription` | Short description shown in the browser. |
| `ServerImageUrl` | URL of an image/icon shown alongside the server name. |
| `UseSsl` | Set to `true` when terminating TLS at the server (requires `cert.pem` and `key.pem` next to the binary). |
| `MaxPlayers` | Maximum players allowed in a single lobby. |
| `ServerSupportsPPModifiers` | Allow players to choose separate gameplay modifiers. |
| `ServerSupportsPPDifficulties` | Allow players to choose their own difficulty. |
| `ServerSupportsPPMaps` | Allow players to play different beatmaps simultaneously (not yet implemented). |
| `MaintenanceStartTime` | Unix timestamp for the start of a maintenance window (`0` = disabled). |
| `MaintenanceEndTime` | Unix timestamp for the end of a maintenance window (`0` = disabled). |
| `LocalizedMessages` | Messages shown to clients during maintenance. `Language` is an integer (0 = English). |
| `RequiredMods` | Minimum mod versions required to connect. Clients with older versions of a listed mod are rejected. |

### `Quickplay`

Controls which level packs appear in the quickplay matchmaking menu.

| Key | Description |
| --- | ----------- |
| `PredefinedPacks` | Built-in Beat Saber pack IDs to expose in quickplay, in display order. |
| `LocalizedCustomPacks` | Custom level packs to expose, with localized display names. `packIds` maps to Beat Saber's internal pack identifiers. |

### `ServerConfiguration`

Controls the master server and dedicated server instancing behaviour.

| Key | Default when unset | Description |
| --- | ------- | ----------- |
| `HostEndpoint` | `127.0.0.1` | The IP address clients connect to for dedicated server instances after hitting the master server's HTTP API, regardless of client configuration. Must be a public IPv4 address for internet-facing servers, a LAN IP for same-network testing, or `127.0.0.1` for fully local setups where both the server and client are on the same machine. IPv6 addresses and domains aren't supported. |
| `UDPBindAddress` | `0.0.0.0` | Local IPv4/IPv6 address the dedicated server instances bind to for UDP traffic. |
| `BasePort` | `30000` | First UDP port used for dedicated server instances. |
| `MaximumSlots` | `10000` | Maximum number of concurrent dedicated server instances. Together with `BasePort` this defines the UDP port range. |
| `SessionTimeToLive` | `180` | Seconds before an idle master server session is expired. |
| `AuthenticateClients` | `true` | When `true`, clients must be authenticated against the Beat Saber platform (Steam/Oculus). Set to `false` for open servers. |
| `AuthBypassAllowed` | `true` | When `AuthenticateClients` is `true`, controls what happens to platforms **not** listed in `AuthedClients`. If `true` (default), unlisted platforms skip verification and connect freely. If `false`, unlisted platforms are **rejected** — effectively making `AuthedClients` an allowlist: only explicitly listed platforms can connect. |
| `AuthedClients` | `[]` | Specifies which platforms must be authenticated against their platform API when `AuthenticateClients` is `true`. Only platforms listed here are verified; unlisted platforms skip authentication and are accepted automatically. An empty array means no platform is verified (everyone passes). Accepted values: `Steam`, `OculusPC`, `OculusQuest`, `PS4`, `PS5`, `Pico`. Example: `["Steam", "OculusQuest"]`. |
| `VersionRanges` | — | List of `{ MinVersion, MaxVersion }` objects. Clients outside every range are rejected. Omit `MaxVersion` to allow all versions above `MinVersion`. |

---

## SSL / TLS

To enable HTTPS, place `cert.pem` and `key.pem` in the same directory as the binary, set `Status.UseSsl` to `true`, and update `Urls` to use `https://`:

```json
{
  "Urls": "https://0.0.0.0:8989",
  "Status": {
    "UseSsl": true
  }
}
```

---

## Client Mod Configuration

Point the BeatTogether mod at your server by editing `BeatTogether.json`:

- **PC:** `BeatSaber/UserData/BeatTogether.json`
- **Quest:** `/sdcard/ModData/com.beatgames.beatsaber/Configs/BeatTogether.json`

```json
{
  "Servers": [
    {
      "ServerName": "My Server",
      "HostName": "<Public IP / Domain>",
      "ApiUrl": "http://<Public IP / Domain>:8989",
      "StatusUri": "http://<Public IP / Domain>8989/status",
      "MaxPartySize": 30
    }
  ]
}
```

Replace `<Public IP / Domain>` with the same IP used for `ServerConfiguration.HostEndpoint` or a domain that points to it. If you used a different port number, replace all instances of `8989` with it.

---

## Troubleshooting

| Symptom | Likely causes |
| ------- | ------------ |
| Clients cannot connect to lobbies | Either `HostEndpoint` is not set to your public IP, your UDP port range isn't forwarded, or `UDPBindAddress` can't be `0.0.0.0`. |
| Clients cannot connect to the instance at all | Either your client config is incorrect or `/status` is unreachable. |
| All players see "server full" | `MaximumSlots` is too low or all ports exhausted |
| Clients on newer game versions are rejected | `VersionRanges` does not include the new version |
| Maintenance message shown unexpectedly | `MaintenanceStartTime` / `MaintenanceEndTime` timestamps are stale — set both to `0` |
