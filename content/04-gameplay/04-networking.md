---
title: "Networking"
---

LAB can run one scene on several machines at once. One process is the **host**: it runs the
game as the authority and, if you like, plays it too. The other processes are **clients** that
join it. Everything travels over UDP (or over Steam, see below), and **scripts decide what is
shared**.

What v1 gives you:

- one server and several clients (8 by default), server authoritative
- entities whose position, rotation and scale follow the authority
- entities a script spawns on the server that appear on every client
- named variables on an entity that everyone can read
- remote function calls (RPC) between the peers
- Steam lobbies and invites, so players can find each other
- all of it from a native C++ module as well as from Lua, see [From C++](#from-c)

What it does not do is listed under [The limits of v1](#the-limits-of-v1). Read that before
designing around it.

## Hosting and joining

### From the command line

`LABRuntime` and `LABEditor` take the same flags:

| Flag | Does |
|---|---|
| `--net-host <port>` | Start a server on that UDP port once the start scene is running |
| `--net-connect <address:port>` | Join a server once the start scene is running. The port defaults to 7777 |
| `--net-bind <address>` | With `--net-host`, listen only on this local IPv4 address instead of every interface. `127.0.0.1` keeps the server on this machine |
| `--net-max-clients <n>` | Cap the number of clients on a server |
| `--log-suffix <name>` | Write `logs/LAB-<name>.log` instead of `logs/LAB.log`, so two processes started from one folder do not share a log |

Host wins if both `--net-host` and `--net-connect` are given. A script that already went online
from `on_create` is left alone.

A dedicated server is `LABRuntime --net-host 7777 --headless`. It still opens a hidden window
and needs a Vulkan device.

**Which addresses a server listens on.** Without `--net-bind` a server listens on every network
interface, which is what a game server that other machines join needs. On Windows that makes the
firewall ask for an exception, once per executable path, so a build in a new folder asks again.
For a session that only ever runs on one machine, add `--net-bind 127.0.0.1`: the socket then
stays on the loopback interface and nothing asks. The same choice exists in Lua as the third
argument of `net.host`, and in the Network panel as **Host on LAN**.

Two windows on one machine:

```bash
LABRuntime --project MyGame.lab --scene scenes/arena.Lscene --net-host 7777 --net-bind 127.0.0.1 --log-suffix host
LABRuntime --project MyGame.lab --scene scenes/arena.Lscene --net-connect 127.0.0.1:7777 --log-suffix client1
```

Every peer must load the same scene. A client that joins learns the server's scene path in the
handshake, but it does not fetch files: the scene and every asset it uses must already be in
the client's copy of the project.

### From the editor: the Network panel

Open **Windows → Editor Settings** and expand **Network** (or **Ctrl+P**, then type "Network"). What it shows depends on
whether you are playing.

**While editing**, networking is off, so the panel holds the settings Play will use:

| Setting | Does |
|---|---|
| **Play as host** | Start a server as soon as Play begins |
| **Port** | The UDP port to host on. Default 7777 |
| **Max clients** | How many clients the server accepts. Default 8 |
| **Host on LAN (all interfaces)** | Off by default: the server listens on `127.0.0.1`, so only this machine can join and Windows does not ask about the firewall. Turn it on when other machines should join. Windows then asks for a firewall exception the first time |
| **Clients to launch on Play** | How many client windows to open, connected to this host |

These are saved with your editor preferences, not in the project, so they survive a restart
and do not reach version control. A script that hosts or connects in `on_create` wins over
**Play as host**.

**While playing**, the panel shows the session and lets you change it:

- the mode (Offline, Server or Client) and the transport (ENet or Steam)
- your client id and the network tick rate
- one row per connected client: id, address, round trip time, bytes sent and received
- totals for the whole session, including packets lost

The controls are **Host** (port, max clients and the same **Host on LAN** choice), **Connect** (`address:port`), **Host via
Steam**, **Connect via Steam** (the host's Steam id) and **Disconnect**. The two Steam buttons
are greyed out, with a tooltip, when Steam is not running. Stopping play disconnects.

### Launching client windows

To try multiplayer on one machine, let the editor open the clients for you.

1. Save the scene. Clients are separate processes and load the **saved file**, not the editor's
   copy, so the button stays disabled while the scene has unsaved changes.
2. Tick **Play as host** and set **Clients to launch on Play**, or press Play and then
   **Launch client** in the Network panel once the editor is hosting.

Each client is a `LABRuntime` started with `--project`, `--scene`, `--net-connect
127.0.0.1:<port>` and `--log-suffix client<N>`, so its log is `logs/LAB-client<N>.log`.
**Launch client** connects to the port shown in the panel, so if a script hosted on a
different port, set the panel to match.

The editor finds `LABRuntime` next to its own folder (`bin/<config>/LABRuntime/`). If you built
only the editor, the button is disabled and its tooltip says so. Build the `LABRuntime` target
too.

Clients are the editor's children. Stopping play closes them, and so does closing the editor,
even if it crashes. A client you close yourself simply disappears from the count. All client
windows open in the same place, so drag them apart.

## Marking entities as networked

Only entities with a **Network Identity** component take part. Add it from **Add Component**.
Everything else stays local to each player.

| Field | Does |
|---|---|
| **Replicate Transform** | Send this entity's position, rotation and scale to the other peers when it moves. Off keeps the entity networked (spawn, variables, RPC) but lets each peer place it |
| **Interpolate** | On peers that do not control the entity, draw it about 100 ms behind the newest update and blend between updates. Off snaps to each update |
| **Transform Authority** | **Server**: the host moves it and everyone follows. **Owner**: the owning client moves it and the host relays that |
| **Allow Client RPC** | Let any client call this entity's `rpc_` functions on the server, not just its owner. For shared things like a game manager or a door. Off by default |

While playing, the component also shows the **Owner** (0 is the server, clients count from 1)
and whether **this peer has authority** over the entity, then one read-only row for each
replicated variable the entity holds (text in quotes, vectors as `(x, y, z)`, entities as
`entity <id>`). The owner and the runtime state are not saved in the scene.

The entity's id in the scene file is its network id, so an entity placed in the editor means
the same thing on every peer with no setup. An entity created at runtime gets its id from the
server.

## The `net` API

Available in editor play and in `LABRuntime`. Call it from any script, including `on_create`.

| Call | Does |
|---|---|
| `net.host(port [, max_clients [, bind_address]])` | Start a server over UDP. Returns `true` if it started. Without `bind_address` it listens on every interface; pass `"127.0.0.1"` to accept only this machine. The address must be an IPv4 number, not a host name |
| `net.connect(address, port)` | Start joining. Returns at once; the result arrives as `on_connected` or `on_disconnected` |
| `net.host_steam([virtual_port, max_clients])` | Start a server over Steam |
| `net.connect_steam(steam_id [, virtual_port])` | Join a host by Steam id (a decimal string) |
| `net.disconnect()` | Leave, or stop hosting |
| `net.is_server()`, `net.is_client()`, `net.is_offline()`, `net.is_connected()` | Where you are |
| `net.transport()` | `"enet"`, `"steam"` or `nil` |
| `net.client_id()` | Your id. 0 on the server |
| `net.clients()` | Server only: an array of connected client ids |
| `net.sender()` | Who sent the RPC or variable change being handled right now |
| `net.spawn(prefab, position [, rotation, owner])` | Server only: create an object on every peer |
| `net.stats()` | `{ rtt_ms, sent_bytes, recv_bytes, packets_lost }` |
| `net.set_tick_rate(hz)` | Host only, before `net.host`: how many snapshots a second (1 to 240, default 30). Clients adopt the host's rate when they join. Returns `false` once a connection exists |
| `net.tick_rate()` | The rate in effect |
| `net.set_interpolation_delay(seconds)` | How far behind the newest snapshot this peer shows other peers' entities, 0 to 0.5 (default 0.1). A shorter delay feels more responsive and wants a higher tick rate; about three ticks is a good value. Takes effect at once and survives a disconnect |
| `net.interpolation_delay()` | The delay in effect |
| `net.client_address(client_id)` | Server only: the address a client connected from, without the port, or `nil`. For refusing a client by where it came from |
| `net.disconnect_client(client_id [, reason])` | Server only: drop one client, who is shown the reason. Returns `false` for an id that is not connected |

And on entities:

| Call | Does |
|---|---|
| `entity:is_networked()` | Whether it has a Network Identity |
| `entity:net_owner()`, `entity:is_owner()` | Who owns it, and whether that is you |
| `entity:has_authority()` | Whether this peer is the one that moves it |
| `entity:set_net_owner(client_id)` | Server only: hand the entity to a client |
| `entity:net_set(key, value)`, `entity:net_get(key)` | Replicated variables |
| `entity:rpc(target, name, ...)`, `entity:rpc_unreliable(target, name, ...)` | Remote calls |

### Callbacks

Define any of these in a script and the engine calls it:

| Callback | Runs on | When |
|---|---|---|
| `on_client_connected(id)` | the server | A client finished joining |
| `on_client_disconnected(id)` | the server | A client left |
| `on_connected()` | a client | The server accepted you |
| `on_disconnected(reason)` | a client | The connection ended or was refused. `reason` is text |
| `on_net_var_changed(key, value)` | that entity's scripts | A replicated variable of that entity changed |

The first four run on every script in the scene. `on_net_var_changed` runs on the scripts of the
entity whose variable changed.

### Spawning

```lua
function on_client_connected(id)
    local player = net.spawn("objects/player.Lobj", vec3.new(0, 0, 1), vec3.new(0, 0, 0), id)
end
```

`net.spawn` works on the server only. It creates the object there, sends the spawn to every
client, and gives the new entity to `owner` (default 0, the server). A client that joins later
is sent every networked object spawned so far. Destroying the entity on the server destroys it
everywhere.

The spawned object needs a Network Identity on its root entity. Only **root entities** are
networked: a networked entity with a parent logs a warning and is skipped.

### Replicated variables

```lua
-- on the authority
entity:net_set("health", 80)

-- on every peer
local health = entity:net_get("health")

function on_net_var_changed(key, value)
    if key == "health" then log.info("health is now " .. value) end
end
```

Values are `nil`, booleans, numbers, strings or `vec3`. Setting a variable to `nil` removes it.
Changes are sent reliably, and a client that joins later receives the current values. Only the
server, or the entity's owner, may write a variable; a write from anyone else is dropped.

There are limits: a key is at most 64 bytes, a string value at most 1024 bytes, and an entity
holds at most 128 variables. A write past them is refused.

Use variables for state that changes now and then (health, score, a door being open). For a
value that changes every frame, let the transform carry it.

### RPC

```lua
-- client, on the player it owns
entity:rpc("server", "rpc_fire", vec3.new(0, 1, 0))

-- a function on that same entity, on the server
function rpc_fire(direction)
    log.info("client " .. net.sender() .. " fired")
end
```

The target is `"server"`, `"clients"` (every client, and the host's own scripts), `"owner"` or
`"all"`. Arguments are up to 8 values of the types above, plus entities. `entity:rpc` is
reliable and ordered; `entity:rpc_unreliable` may be lost and is for things sent every frame.

The rules that keep a joined stranger from running your code:

- **Only functions whose name starts with `rpc_` can be called remotely.** The prefix is the
  opt-in. A peer cannot call `on_update`, `on_create` or any helper by name. The name must be
  letters, digits and underscores, up to 64 bytes. Anything else is dropped and logged.
- The receiver calls the function on **every script attached to that entity**, with
  `net.sender()` holding the original sender's id, even when the server relayed the call.
- The server accepts a client's RPC for an entity **that client owns**, or for any entity with
  **Allow Client RPC** on. A client accepts RPCs only from the server.
- Treat every argument as untrusted input from a stranger. Check ranges before using them.

## Visual scripting

The Node Graph has a **Network** category with nodes for all of the above. They compile to the
same `net.*` calls a hand-written script makes. See [Visual Scripting](../06-scripting/01-visual-scripting.md)
for the editor itself.

| Kind | Nodes |
|---|---|
| Events | On Client Connected, On Client Disconnected, On Connected, On Disconnected, On Net Var Changed, On Lobby Entered, On Lobby Join Requested |
| Session | Host, Connect, Host Steam, Connect Steam, Disconnect |
| Queries | Is Server, Is Client, Is Connected, Client Id, Is Owner, Has Authority |
| Entities | Net Spawn, Set Net Owner |
| Variables | Net Set and Net Get, each in a Number, Bool, Text and Vector form |
| Calls | RPC |

A few things are worth knowing:

- **Variables come in four types** because a node's pins have fixed types. Pick the one that
  matches what you store. **On Net Var Changed** has one output per type; wire the one you
  expect.
- **Host and Connect** have a **Port** pin with a value typed into it. **Host** leaves the
  max-clients argument out when it is 0, so the default of 8 applies.
- **Connect** and **Connect Steam** take the address or Steam id either from a wire or from the
  text typed into the node.
- **RPC** sends the **Arg0** to **Arg3** pins, which are a number, a vector, a text and an
  entity, stopping at the last one you connected or set. Name the function in the node; it
  must start with `rpc_`. The receiving function is ordinary Lua.
- The nodes are the sending side only. A function a remote call runs is a Lua script.

## Steam lobbies and invites

A **lobby** is how players find each other. The game then connects to the lobby's owner over
the Steam transport. All of it needs the game to be running under Steam, and every call is a
harmless no-op (returning `false` or `nil`) when it is not. See [`docs/STEAM.md`](../STEAM.md)
for the plumbing.

The `steam_lobby` table:

| Call | Does |
|---|---|
| `steam_lobby.create(type, max_members)` | `"public"`, `"friends"`, `"private"` or `"invisible"`. Asynchronous |
| `steam_lobby.find({ string = {..}, number = {..}, slots = n, distance = "..", max_results = n })` | Search for lobbies. Asynchronous |
| `steam_lobby.join(id)`, `leave()`, `current()`, `owner()` | Move in and out, and ask who runs it |
| `steam_lobby.set_data(k, v)`, `get_data(k [, id])`, `members()` | Searchable lobby fields and the member list |
| `steam_lobby.set_member_data(k, v)`, `get_member_data(steam_id, k)` | Per-player fields |
| `steam_lobby.invite_dialog()`, `invite(steam_id)`, `set_joinable(b)`, `send_chat(text)` | Invites and chat |

Lobby ids and Steam ids are **decimal strings**: they are 64-bit and do not fit a Lua number.

Callbacks: `on_lobby_created(ok, id)`, `on_lobby_entered(ok, id)`, `on_lobby_list(results)`,
`on_lobby_member_joined(steam_id)`, `on_lobby_member_left(steam_id)`,
`on_lobby_data_changed()`, `on_lobby_chat(steam_id, text)` and
`on_lobby_join_requested(id)`.

### From lobby to game

The owner hosts and everyone else connects to the owner. The one thing to get right is
**order**: a member must not connect before the owner is listening. The pattern is a lobby
field the owner sets once it is ready, here called `host_ready` (the name is your choice):

```lua
-- owner
function on_lobby_entered(ok, id)
    if ok and steam_lobby.owner() == steam_lobby.user_id() then
        net.host_steam()
        steam_lobby.set_data("host_ready", "1")
    end
end

-- members
function on_lobby_data_changed()
    if steam_lobby.get_data("host_ready") == "1" and not net.is_connected() then
        net.connect_steam(steam_lobby.owner())
    end
end
```

A member that joins after `host_ready` is already set gets `on_lobby_data_changed` on entry
and connects straight away. Once the game has started and the lobby is no longer needed, call
`steam_lobby.leave()`; Steam asks that lobbies are not kept around and their data is not used
as a state stream.

### Invites

`steam_lobby.invite_dialog()` opens the Steam overlay's invite list. When a friend accepts:

- **the game is running**: `on_lobby_join_requested(id)` runs. It does not join for you, because
  you may need to leave a session first or ask the player. Call `steam_lobby.join(id)` when
  ready.
- **the game is not running**: Steam starts it with `+connect_lobby <id>` (the plus sign is
  Steam's spelling), and the game joins that lobby once the start scene is running, which
  raises `on_lobby_entered`.

## From C++

A native gameplay module (see [Native Scripting](../NATIVE_SCRIPTING.md)) can do everything the
`net` and `steam_lobby` tables do, with the same rules, because both go through the same code in
the engine. It needs an engine at ABI minor 11 or newer, and a module built against an older header
still loads and runs, it just has no networking.

Include `LABNative.hpp`. The calls are free functions in `LABNative::Net` and
`LABNative::SteamLobby`, the entity ones also exist on `Behaviour` with the entity filled in, and
values are `LAB_Variant`s made with `LABNative::Variant`. Steam ids and lobby ids are `uint64_t`
in C++, so unlike Lua there is no string to convert.

| Lua | C++ |
|---|---|
| `net.host(port, max, address)`, `net.connect(address, port)` | `Net::Host(port, max, address)`, `Net::Connect(address, port)` |
| `net.host_steam`, `net.connect_steam(id)` | `Net::HostSteam`, `Net::ConnectSteam(id)` |
| `net.is_server()`, `net.client_id()`, `net.clients()` | `Net::IsServer()`, `Net::GetClientId(id)`, `Net::Clients()` |
| `net.sender()`, `net.stats()` | `Net::GetSender(id)`, `Net::GetStats(stats)` |
| `net.set_tick_rate(hz)`, `net.tick_rate()` | `Net::SetTickRate(hz)`, `Net::GetTickRate()` (ABI minor 13) |
| `net.set_interpolation_delay(s)`, `net.interpolation_delay()` | `Net::SetInterpolationDelay(s)`, `Net::GetInterpolationDelay()` (minor 13) |
| `net.client_address(id)`, `net.disconnect_client(id, reason)` | `Net::GetClientAddress(id)`, `Net::DisconnectClient(id, reason)` (minor 13) |
| `net.spawn(prefab, position, rotation, owner)` | `Net::Spawn(prefab, position, rotation, owner)` |
| `e:net_set(key, value)`, `e:net_get(key)` | `NetSet(key, value)`, `NetGet(key)` |
| `e:rpc("clients", "rpc_name", ...)` | `Rpc(Net::Target::Clients, "rpc_name", { ... })` |
| `function rpc_name(...)` | `OnRpc(name, args, argc, sender)` |
| `on_client_connected(id)` and the other net callbacks | `OnClientConnected(id)`, `OnClientDisconnected(id)`, `OnConnected()`, `OnDisconnected(reason)`, `OnNetVarChanged(key, value)` |
| `steam_lobby.*` and its `on_lobby_*` callbacks | `SteamLobby::*` and `OnLobbyCreated`, `OnLobbyEntered`, `OnLobbyList`, `OnLobbyMemberJoined`, `OnLobbyMemberLeft`, `OnLobbyDataChanged`, `OnLobbyChat`, `OnLobbyJoinRequested` |

A few differences from Lua are worth knowing:

- **One `OnRpc` receives every call.** The engine has already checked that the name starts with
  `rpc_` and holds only letters, digits and underscores, and that the sender may call it, so the
  behaviour compares the name and ignores the ones it does not know. Calls from the network reach
  the Lua scripts on the entity as well as its native behaviours, Lua first.
- **Connection and lobby events reach every behaviour**, wherever it sits, so one manager entity
  is enough. A variable change and an RPC reach only the behaviours on that entity.
- **An exception in `OnRpc` does not disable the behaviour.** Any permitted remote caller can
  cause one. It is recorded once per function. An exception in any other callback disables the
  behaviour, as in `OnUpdate`.
- **Finding lobbies** adds filters and then searches, the shape Steam uses:
  `SteamLobby::AddStringFilter("mode", "coop")`, `SteamLobby::AddNumberFilter("level", 3,
  LAB_LOBBY_COMPARE_GREATER_EQUAL)`, `SteamLobby::Find()`. The results arrive in
  `OnLobbyList` as an array of `LAB_LobbyInfo` (id, members, maximum). Read a result's searchable
  data with `SteamLobby::GetData(key, lobbyId)`.
- With no Steam, every lobby call answers false, 0 or an empty string and nothing else happens.

A small match manager. The entity runs this behaviour on the server and on every client. The host
is started the usual way (`--net-host 7777` on the command line, or `Net::Host` from a menu), and
the client-side callbacks below simply never fire on the server:

```cpp
#include "LABNative.hpp"
#include <cstring>

class Match : public LABNative::Behaviour
{
public:
    // Server only: every behaviour hears a client arrive.
    void OnClientConnected(uint32_t id) override
    {
        LABNative::Net::Spawn("objects/player.Lobj", LAB_Vec3{ 0, 0, 2 }, LAB_Vec3{}, id);
        NetSet("players", LABNative::Variant::Number((double)LABNative::Net::Clients().size()));
        Rpc(LABNative::Net::Target::Clients, "rpc_welcome", { LABNative::Variant::Text("hello") });
    }

    // A client called manager:rpc("server", "rpc_goal", 1) on this entity.
    void OnRpc(const char* name, const LAB_Variant* args, int argc, uint32_t sender) override
    {
        if (std::strcmp(name, "rpc_goal") == 0)
        {
            const double score = NetGet("score").AsNumber() + LABNative::Net::ArgNumber(args, argc, 0);
            NetSet("score", LABNative::Variant::Number(score));
        }
    }

    // Everywhere else, when the score changes.
    void OnNetVarChanged(const char* key, const LAB_Variant& value) override
    {
        if (std::strcmp(key, "score") == 0)
            LABNative::LogInfo("the score changed");
    }
};
```

The entity needs a `NetworkIdentityComponent` with an `IDComponent`, as for Lua, and
**Allow Client RPC** ticked for the client call above to be accepted.

## The limits of v1

Design around these:

- **No client-side prediction.** A client's own movement is what the server says it is, after
  a round trip. On a good connection that is fine for a cooperative or slow game and poor for a
  twitch shooter. The exception is Owner authority, where the owning client moves an entity
  itself and the server accepts it, which trades cheating protection for responsiveness.
- **No reconciliation, rollback or lag compensation.**
- **Root entities only.** A networked entity with a parent is skipped with a warning.
- **Physics runs on the authority only.** On everyone else a replicated physics body is driven
  by the received transform as a kinematic body, so it does not push things locally.
- **Interpolation does not extrapolate.** A peer holds the last transform when updates stop.
- **No NAT punch-through.** Hosting over UDP on the internet needs a forwarded port. Steam
  hosting does not.
- **No encryption** on the UDP transport. Steam's transport is encrypted by Steam.
- **Clients do not download assets.** Every peer needs the same project files.
- **A replicated variable holds one value**, not a table. Split a record into several keys.
- **Scene changes are minimal.** When the server opens another scene, clients follow and the
  networked spawns are sent again. Anything richer than that is yours to handle.
- **A dedicated server still needs a Vulkan device** and opens a hidden window.
- **Lobbies are the only matchmaking.** There is no server browser over UDP and no Steam game
  server support.
