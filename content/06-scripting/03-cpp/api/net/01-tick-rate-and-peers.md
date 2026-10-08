---
title: "Networking: tick rate, interpolation delay and peers"
---

The networking entries added at ABI minor 13. The rest of `api->Net` (hosting, joining, spawning,
variables, RPCs) is in [`docs/NATIVE_SCRIPTING.md`](../../../../NATIVE_SCRIPTING.md) under
"Networking" and in [Networking](../../../../04-gameplay/04-networking.md); these six exist for a game that
needs a tight client and a host that vets who joins.

All six are appended to the end of `LAB_NetAPI`, and on an engine older than minor 13 the
wrappers answer false, the defaults, or an empty string.

```c
int (*SetTickRate)(float ticksPerSecond);
float (*GetTickRate)(void);
int (*SetInterpolationDelay)(float seconds);
float (*GetInterpolationDelay)(void);
int (*GetClientAddress)(uint32_t client, char* buffer, size_t capacity);
int (*DisconnectClient)(uint32_t client, const char* reason);
```

```cpp
bool LABNative::Net::SetTickRate(float ticksPerSecond);
float LABNative::Net::GetTickRate();
bool LABNative::Net::SetInterpolationDelay(float seconds);
float LABNative::Net::GetInterpolationDelay();
std::string LABNative::Net::GetClientAddress(uint32_t client);
bool LABNative::Net::DisconnectClient(uint32_t client, const char* reason = "disconnected");
```

## `SetTickRate` / `GetTickRate`

How many times a second the network sends snapshots. The default is 30. **A host sets it before
`Host`**: the rate is fixed for a session, so the call answers false once a connection exists, and
for a rate outside 1 to 240. The Welcome carries the host's rate to each client, which adopts it and
ignores its own value, so only the host's call counts; `GetTickRate` on a client reads the rate it
adopted.

```cpp
LABNative::Net::SetTickRate(60.0f);                       // before Host
LABNative::Net::Host(7777, 1, "127.0.0.1");
```

A higher rate means smaller gaps between snapshots, so less latency from a host's physics step to
a client's screen, at the cost of bandwidth. It is independent of the physics rate, which stays at
60 steps a second.

## `SetInterpolationDelay` / `GetInterpolationDelay`

How far behind the newest snapshot a client draws other peers' entities, in seconds, clamped to 0
through 0.5; the default is 0.1. The client always shows the past by this much so that it has two
samples to blend between: three ticks is a good value (0.05 at 60 Hz), and the delay is usually the
largest single part of a client's input-to-motion latency. It is the client's own setting, takes
effect at once in any state, survives a disconnect, and a value that is not a number is refused.
Zero shows each snapshot as it arrives, with no blending.

## `GetClientAddress`

Server only. The address a connected client came from, as a bare IPv4 literal with no port
(`"192.168.1.4"`). The wrapper returns it as a string, empty for an unknown client, on a client,
and over the Steam transport, which has no address to give. Use it from `OnClientConnected` to
refuse a peer that connected from outside the network the game is meant for.

## `DisconnectClient`

Server only. Drops one client, who is shown `reason` (a client dropped with no reason reads
"disconnected"), and the server then sees `OnClientDisconnected` for it like any other departure.
False for an unknown client and when this peer is not a server.

```cpp
void OnClientConnected(uint32_t client) override
{
	if (!IsPrivateAddress(LABNative::Net::GetClientAddress(client)))
		LABNative::Net::DisconnectClient(client, "wrong network");
}
```
