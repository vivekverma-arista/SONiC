# Event-Driven Switchover on Link Failure

## Table of Contents

- [Overview](#overview)
- [Background: Existing Graceful Switchover Protocol](#background-existing-graceful-switchover-protocol)
- [Goals](#goals)
- [Non-Goals](#non-goals)
- [Design](#design)
  - [Centralized TlvCommandMonitor](#centralized-tlvcommandmonitor)
  - [Protocol Detail: Port Identification via `echo.id`](#protocol-detail-port-identification-via-echoid)
  - [Packet flow](#packet-flow)
  - [ICMP packet format](#icmp-packet-format)
  - [Packet acceptance filter (receive side)](#packet-acceptance-filter-receive-side)
- [Configuration](#configuration)
- [Component Changes](#component-changes)
  - [New: `sonic-linkmgrd/src/link_prober/TlvCommandMonitor.h/.cpp`](#new-sonic-linkmgrdsrclink_probertlvcommandmonitorhcpp)
  - [Modified: `sonic-linkmgrd/src/DbInterface.h/.cpp`](#modified-sonic-linkmgrdsrcdbinterfacehcpp)
  - [Modified: `sonic-linkmgrd/src/MuxManager.h/.cpp`](#modified-sonic-linkmgrdsrcmuxmanagerhcpp)
  - [Modified: `sonic-linkmgrd/src/MuxPort.h/.cpp`](#modified-sonic-linkmgrdsrcmuxporthcpp)
  - [Modified: `sonic-linkmgrd/src/link_manager/LinkManagerStateMachineActiveStandby.h/.cpp`](#modified-sonic-linkmgrdsrclink_managerlinkmanagerstatemachineactivestandbyhcpp)
- [Testing](#testing)
  - [Unit tests (`make test`)](#unit-tests-make-test)
  - [sonic-mgmt integration test](#sonic-mgmt-integration-test)

## Overview

In the current dual-ToR implementation, switchover convergence when the active ToR loses its link to the server is bounded by the ICMP heartbeat probe interval (default 100ms). The active ToR switches the MUX to standby over I2C, but the standby ToR only discovers this on the next probe cycle when it receives its own GUID back instead of the peer's. This makes the standby-side reaction poll-driven.

```
                        T1 Spine Fabric
          +------------------------------------------+
          |                                          |
    +-----+------+                          +--------+-----+
    |   ToR A    | <---- Loopback0/BGP ----> |    ToR B    |
    |  (active)  |       T1 uplinks          |  (standby)  |
    +-----+------+                          +--------+-----+
          |                                          |
          |          +------------------+            |
          +--------> |   Hardware MUX   | <----------+
                     |  (selects active)|
                     +--------+---------+
                              |
                           Server NIC
```

This document describes making the switchover fully event-driven: when the active ToR detects its own link failure, it immediately notifies the standby via a direct ICMP message over the T1 uplinks, allowing the standby to begin its MUX takeover without waiting for the next probe.

## Background: Existing Graceful Switchover Protocol

linkmgrd already implements a graceful switchover protocol for planned transitions (e.g., `config mux mode standby` on the active ToR). The mechanism works as follows:

1. The active ToR embeds a `COMMAND_SWITCH_ACTIVE` TLV in its regular ICMP heartbeat sent toward the server.
2. The server echoes the packet back. Both ToRs receive the echo reply.
3. The standby ToR's `LinkProberBase::handleTlvCommandRecv()` parses the TLV and fires a `SwitchActiveRequestEvent` on its state machine.

The wire format is already defined in `sonic-linkmgrd/src/link_prober/IcmpPayload.h`:

```
IcmpPayload {cookie, version, uuid[8], seq}   ← 24 bytes
TLV         {type=TLV_COMMAND(0x5), length=1, command=COMMAND_SWITCH_ACTIVE(1)}
TLV         {type=TLV_SENTINEL(0xff), length=0}
```

This protocol is **not triggered on link failure** today — it requires the server's echo for delivery, which stops working when the server link is down. The new design sends the notification directly to the peer ToR over T1, bypassing the server path entirely.

## Goals

- Reduce standby-side convergence latency on link failure from ~1 probe interval to network RTT (T1 path).
- Reuse the existing `COMMAND_SWITCH_ACTIVE` TLV wire format with no protocol changes.
- Support any number of MUX ports with no hardcoded interface names.

## Non-Goals

- Active-Active cable type: out of scope for this change.
- Guaranteed delivery: the notification is best-effort over UDP/ICMP. The existing probe-based switchover remains the fallback if the notification is lost.

## Design

### Centralized TlvCommandMonitor

A single new class, `TlvCommandMonitor`, is owned by `MuxManager` (one instance per daemon). It opens **one `AF_INET SOCK_RAW IPPROTO_ICMP` socket** for both send and receive. Packets are sent to and received from the **peer's Loopback0 IP** (from `CONFIG_DB.PEER_SWITCH`). The kernel routes the packets through whichever T1 PortChannels are available, so the implementation is agnostic to the number and names of uplinks.

Port identity within a received packet is carried by the ICMP `echo.id` field, which already equals `htons(serverId)` in the existing probing protocol (`sonic-linkmgrd/src/link_prober/LinkProberBase.cpp:369`). On receive, `TlvCommandMonitor` extracts `echo.id`, looks up the matching `MuxPort` in `MuxManager::mPortMap`, and posts a `SwitchActiveRequestEvent` to that port's per-port strand.

### Protocol Detail: Port Identification via `echo.id`

Each MUX port in linkmgrd has a numeric `serverId` derived from the port name at daemon startup (`sonic-linkmgrd/src/MuxManager.cpp:492`):

```cpp
uint16_t serverId = atoi(portName.substr(portName.find_last_not_of("0123456789") + 1).c_str());
```

This strips the trailing decimal digits from the interface name: `Ethernet0` → `0`, `Ethernet4` → `4`, `Ethernet108` → `108`. The `serverId` is stored in `MuxPortConfig` and is stable for the lifetime of the port object.

The existing ICMP probing protocol already embeds `serverId` in every heartbeat packet sent toward the server (`sonic-linkmgrd/src/link_prober/LinkProberBase.cpp:369`):

```cpp
icmpHeader->un.echo.id = htons(mMuxPortConfig.getServerId());
```

The `COMMAND_SWITCH_ACTIVE` packet sent over T1 reuses this same `icmphdr` field verbatim:

```
icmphdr.type        = ICMP_ECHOREPLY (0)
icmphdr.code        = 0
icmphdr.un.echo.id  = htons(serverId)   ← identifies the target MUX port
icmphdr.un.echo.seq = 0
```

The field is 16 bits, so up to 65535 ports can be addressed — well beyond any realistic deployment.

**Receive-side dispatch:**

When `TlvCommandMonitor::handleRecv()` accepts a packet (all three filters passed), it extracts the target port:

```cpp
uint16_t serverId = ntohs(icmpHeader->un.echo.id);
```

`MuxManager::handleSwitchCommandForPort(serverId)` iterates `mPortMap` and compares `port->getServerId()` against the extracted value. The map is bounded by the number of MUX ports (typically ≤ 128), so the linear scan is negligible. No additional discriminator is needed: `echo.id` is set only by linkmgrd and matches exactly one port per daemon instance.

```
  Received ICMP packet
          |
          v
  +-------+-------------------------------------------+
  |       TlvCommandMonitor::handleRecv()             |
  |                                                   |
  |  icmphdr.type == ICMP_ECHOREPLY?  --No--> discard |
  |  icmpPayload.cookie == 0x47656d69? -No--> discard |
  |  iphdr.saddr == peer Loopback0?   --No--> discard |
  |                                                   |
  |  serverId = ntohs(icmphdr.un.echo.id)             |
  +-------+-------------------------------------------+
          |
          v
  MuxManager::handleSwitchCommandForPort(serverId)
          |
          +-- serverId == 0 --> Ethernet0
          +-- serverId == 4 --> Ethernet4    (MuxPort::handleSwitchActiveRequestEvent)
          +-- serverId == 8 --> Ethernet8
          +-- ...
          |
          v  (posted to per-port strand)
  LinkManagerStateMachineActiveStandby::handleSwitchActiveRequestEvent()
          |
          v
  switchMuxState(TlvSwitchActiveCommand, Active)
```

### Packet flow

**Send path (active ToR on link failure):**

```
LinkManagerStateMachineActiveStandby::handleStateChange()
  detects LinkUp→LinkDown with MUX != Standby
  → switchMuxState(LinkDown, Standby)            ← existing: writes to APP_DB; orchagent drives hardware
  → mSendPeerNotifyFnPtr()                       ← NEW: fires TlvCommandMonitor::sendSwitchCommand(serverId)
       → AF_INET SOCK_RAW sendto(peer Loopback0 IP)
            ICMP ECHO_REPLY, echo.id=serverId
            src IP = our Loopback0 (kernel selects via RM_SET_SRC BGP route-map)
            IcmpPayload{cookie=mSoftwareCookie} + TLV{COMMAND_SWITCH_ACTIVE} + TLV_SENTINEL
            kernel routes via T1 uplinks (BGP)
```

**Receive path (standby ToR):**

```
TlvCommandMonitor::handleRecv()
  filter: ICMP type=ECHO_REPLY, cookie=mSoftwareCookie, src IP=peer Loopback0
  extract serverId = ntohs(icmpHeader->un.echo.id)
  → MuxManager::handleSwitchCommandForPort(serverId)
    → MuxPort::handleSwitchActiveRequestEvent()
      → post to per-port strand
        → LinkManagerStateMachineActiveStandby::handleSwitchActiveRequestEvent()
          → switchMuxState(TlvSwitchActiveCommand, Active)
```

### ICMP packet format

`AF_INET SOCK_RAW IPPROTO_ICMP` does not include the Ethernet header. The kernel delivers the full IP packet into the receive buffer and prepends the IP header automatically on send.

```
RX buffer (AF_INET raw socket — kernel includes IP header):

  offset  0  +---------------------------+
             |       IP Header           |  20 bytes
             |   saddr = peer Loopback0  |  <-- source IP filter applied here
  offset 20  +---------------------------+
             |      ICMP Header          |  8 bytes
             |  type = ECHO_REPLY (0)    |
             |  echo.id = serverId (NBO) |  <-- port discriminator
  offset 28  +---------------------------+
             |      IcmpPayload          |  24 bytes
             |  cookie = 0x47656d69      |  <-- software cookie filter applied here
             |  version, uuid[8], seq    |
  offset 52  +---------------------------+
             |  TLV: COMMAND_SWITCH_ACTIVE (type=0x05, len=1, value=1)  |
             +---------------------------+
             |  TLV: SENTINEL (type=0xff, len=0)                        |
             +---------------------------+

TX buffer (sendto — kernel prepends IP header; source IP = our Loopback0 via RM_SET_SRC route-map; routed via BGP to peer Loopback0):

  offset  0  +---------------------------+
             |      ICMP Header          |  8 bytes
             |  type = ECHO_REPLY (0)    |
             |  echo.id = serverId (NBO) |
  offset  8  +---------------------------+
             |      IcmpPayload          |  24 bytes
             |  cookie = 0x47656d69      |
             |  version, uuid[8], seq    |
  offset 32  +---------------------------+
             |  TLV: COMMAND_SWITCH_ACTIVE (type=0x05, len=1, value=1)  |
             +---------------------------+
             |  TLV: SENTINEL (type=0xff, len=0)                        |
             +---------------------------+
```

`ECHO_REPLY` (type=0) is used instead of `ECHO_REQUEST` to avoid the kernel auto-replying to the packet on the receiving side.

### Packet acceptance filter (receive side)

Three conditions must all be true to process a received packet:

1. `icmpHeader->type == ICMP_ECHOREPLY`
2. `ntohl(icmpPayload->cookie) == IcmpPayload::getSoftwareCookie()` — distinguishes from hardware prober and unrelated ICMP traffic
3. `ipHeader->saddr == mPeerLoopback0Ip` — accepts only from the configured peer

The source IP check works because bgpcfgd installs a zebra route-map (`RM_SET_SRC`) that sets Loopback0 as the preferred source address for all BGP-learned routes. Since the route to the peer's Loopback0 is BGP-learned, the kernel automatically uses our own Loopback0 as the packet source — matching exactly what the peer filters on.

Any packet failing these checks is silently dropped and the receive loop restarts.

## Configuration

No new configuration knobs are introduced. The feature reads existing CONFIG_DB entries:

| Table | Key | Field | Used for |
|---|---|---|---|
| `PEER_SWITCH` | `<peer hostname>` | `address_ipv4` | destination IP for send; source IP filter for receive |

`DbInterface::getPeerSwitchInfo()` (new method, follows `getLoopback2InterfaceInfo()` pattern) reads this table once at startup and passes the IP to `MuxManager::setPeerLoopback0Ipv4Address()`, which initializes `TlvCommandMonitor` and starts the receive loop.

If `PEER_SWITCH` is absent from CONFIG_DB (e.g., single-ToR deployment), `getPeerSwitchInfo()` logs a warning and `TlvCommandMonitor` is not initialized. The feature is silently disabled and the daemon operates as before.

## Component Changes

### New: `sonic-linkmgrd/src/link_prober/TlvCommandMonitor.h/.cpp`

Core class. Owns the raw socket, async receive loop (via `boost::asio::posix::stream_descriptor` + per-instance strand), and the send path. Added to `sonic-linkmgrd/src/link_prober/subdir.mk`.

### Modified: `sonic-linkmgrd/src/DbInterface.h/.cpp`

`getPeerSwitchInfo()` reads `PEER_SWITCH` from CONFIG_DB and calls `MuxManager::setPeerLoopback0Ipv4Address()`. Called in the startup config-read sequence at `sonic-linkmgrd/src/DbInterface.cpp:1815`, after `getLoopback2InterfaceInfo()`.

### Modified: `sonic-linkmgrd/src/MuxManager.h/.cpp`

- Owns `mTlvCommandMonitorPtr` (created in constructor, initialized from `setPeerLoopback0Ipv4Address()`).
- `handleSwitchCommandForPort(uint16_t serverId)`: iterates `mPortMap` to find the matching port and calls `MuxPort::handleSwitchActiveRequestEvent()`.
- `getMuxPortPtrOrThrow()`: after creating a port, calls `port->setLoopbackSwitchFn(bind(&TlvCommandMonitor::sendSwitchCommand, serverId))`.

### Modified: `sonic-linkmgrd/src/MuxPort.h/.cpp`

- `getServerId()`: exposes `mMuxPortConfig.getServerId()` for dispatch lookup.
- `handleSwitchActiveRequestEvent()`: posts `LinkManagerStateMachineBase::handleSwitchActiveRequestEvent()` to the per-port strand. This method is already declared `virtual` in `LinkManagerStateMachineBase` and overridden in `ActiveStandbyStateMachine` — no base class changes needed.
- `setLoopbackSwitchFn()` / `getLoopbackSwitchFn()`: stores the `boost::function<void()>` bound by `MuxManager`.

### Modified: `sonic-linkmgrd/src/link_manager/LinkManagerStateMachineActiveStandby.h/.cpp`

- `mSendPeerNotifyFnPtr`: new `boost::function<void()>` alongside the existing `mSendPeerSwitchCommandFnPtr`.
- Bound in `handleSwssBladeIpv4AddressUpdate()` via `mMuxPortPtr->getLoopbackSwitchFn()`.
- Called in `handleStateChange()` immediately after `switchMuxState(LinkDown, Standby)`.

## Testing

### Unit tests (`make test`)

`TlvCommandMonitor` requires a real raw socket and cannot be instantiated in unit tests. The dispatch logic is tested by bypassing the socket entirely and calling `MuxPort::handleSwitchActiveRequestEvent()` directly:

**`sonic-linkmgrd/test/LinkManagerStateMachineTest.cpp`**

`TEST_F(LinkManagerStateMachineTest, ActiveStandbyLoopbackNotify_SwitchesActive)`
- Initial composite state: `(Standby, Standby, Up)`
- Call `mFakeMuxPort.handleSwitchActiveRequestEvent()`
- `runIoService(1)` to drain the posted handler
- Assert `ms(compositeState) == MuxState::Wait`
- Post `MuxState::Active` ack, `runIoService(1)`
- Assert final state `(Active, Active, Up)`

`TEST_F(LinkManagerStateMachineTest, ActiveStandbyLoopbackNotify_WrongCookie_Ignored)`
- Directly call `TlvCommandMonitor::handleRecv()` with a crafted `mRxBuffer` containing a wrong cookie
- Assert `MuxManager::handleSwitchCommandForPort()` is never invoked

**`sonic-linkmgrd/test/MuxManagerTest.cpp`**

`TEST_F(MuxManagerTest, handleSwitchCommandForPort_DispatchesToCorrectPort)`
- Create `Ethernet0` (serverId=0) and `Ethernet4` (serverId=4) in `mPortMap`
- Call `mMuxManagerPtr->handleSwitchCommandForPort(0)`
- `pollIoService(N)` to drain
- Assert only `Ethernet0`'s `handleSwitchActiveRequestEvent` counter incremented

`TEST_F(MuxManagerTest, handleSwitchCommandForPort_UnknownServerId_NoOp)`
- Call with serverId=99, assert no crash and no port state change

**Test stubs needed:**

`sonic-linkmgrd/test/FakeDbInterface`: add `getPeerSwitchInfo()` no-op override (raw socket not available in tests).

`sonic-linkmgrd/test/FakeMuxPort`: add `mHandleSwitchActiveRequestEventCount` counter incremented in `handleSwitchActiveRequestEvent()` for dispatch verification.

### sonic-mgmt integration test

**File:** `tests/dualtor_io/test_link_failure_switchover.py`  
**Topology:** `dualtor`

The simulated environment does not faithfully reproduce BGP routing latency or packet loss, so the tests do **not** assert convergence time or dropped packet counts. The correct invariant to test is whether the TLV packet physically ingressed on the standby DUT and was processed by linkmgrd.

**`test_event_driven_switchover_on_link_failure`**

1. Seed state: upper ToR active, lower ToR standby.
2. Record journalctl cursor on standby before the event.
3. Shut down the MUX-facing interface on the active ToR: `config interface shutdown Ethernet0`.
4. **Primary assertion**: poll journalctl on standby for `TlvCommandMonitor.*COMMAND_SWITCH_ACTIVE.*serverId 0` within 10 seconds. This log line is emitted only when the ICMP packet arrives at the raw socket and passes all filters — it proves end-to-end packet delivery through the T1 path.
5. **Secondary assertion**: verify standby's `show mux status` shows `active` within 30 seconds.
6. Restore the interface.

The primary assertion failing with no log line points specifically at the T1 routing path (Loopback0 → BGP → peer), not at a timing threshold.

**`test_no_spurious_switchover_on_wrong_cookie`**

- Record journalctl cursor on standby.
- Inject a crafted ICMP ECHO_REPLY via scapy/ptf with `cookie=0xdeadbeef` to the standby's Loopback0 IP.
- Wait 2 seconds.
- Assert no `COMMAND_SWITCH_ACTIVE` log appeared and MUX state remains `standby`.

**Fixtures reused from existing infrastructure:**

`upper_tor_host`, `lower_tor_host`, `run_icmp_responder`, `run_garp_service`, `toggle_all_simulator_ports_to_upper_tor`, `wait_until`, `pytest_assert`
