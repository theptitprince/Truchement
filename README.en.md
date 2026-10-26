<div align="center">

# Truchement — Synology → UGOS NUT relay

**One UPS on the Synology, the UGREEN warned too.**

![Version](https://img.shields.io/badge/version-1.1.0-2ea44f)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![NUT](https://img.shields.io/badge/protocol-NUT%20%2F%203493-0a7ea4)
![Platform](https://img.shields.io/badge/platform-Synology%20DSM%20%E2%86%92%20UGREEN%20UGOS-4a4a4a)
![License](https://img.shields.io/badge/license-GPL--3.0--or--later-blue)

</div>

[Français](README.md) · **English**

---

> [!NOTE]
> **Truchement needs its own IP address on the local network** (192.168.1.210 in the example),
> free and outside the router's DHCP range: this address, not the Synology's, is the one you give
> to UGOS. On the DSM side, the address to allow in the network UPS server is **172.31.93.2**, the
> relay's address on the internal Docker network. The addresses in the file are examples: adapt
> the ones marked « à adapter » (“to adapt”).

Truchement lets a UGREEN NAS (UGOS) read the UPS plugged by USB into a
Synology. DSM can share its UPS over the network (NUT server, port 3493),
and UGOS can listen to a network UPS; but the two don't understand each
other: UGOS asks for the UPS `ups0` with its own credentials, while DSM only
knows `ups`, with the fixed credentials `monuser` / `secret`.

Truchement sits between them as an interpreter (*truchement* is an old
French word for one, still alive in *par le truchement de*, “by means of”):
it renames the UPS both ways, substitutes the right credentials, and lets
everything else through untouched. When the power fails, UGOS sees the UPS
go on battery (`OB`), then low battery (`LB`) or forced shutdown (`FSD`),
exactly as the Synology does.

Since 1.1.0, Truchement can also **wake the UGREEN NAS with Wake-on-LAN**
when the power comes back after it has already shut down: otherwise it would
stay off (see [6. Waking the UGREEN NAS](#6-waking-the-ugreen-nas-wake-on-lan)).

```
 UGREEN (UGOS)                    TRUCHEMENT                       SYNOLOGY (DSM)
 NUT client        ----------->   192.168.1.210:3493   ---------->  NUT server
 "ups0"                           (macvlan, LAN)                    "ups", USB
 UGOS credentials                 172.31.93.2  ── internal net ──>  172.31.93.1:3493
                   <-----------   ups ↔ ups0           <----------  monuser / secret
```

Current version: 1.1.0 — the release history is in
[CHANGELOG.md](CHANGELOG.md) (in French).

## Contents

- [1. Requirements](#1-requirements)
- [2. The files](#2-the-files)
- [3. Installation](#3-installation)
- [4. Settings](#4-settings)
- [5. How it works](#5-how-it-works)
- [6. Waking the UGREEN NAS (Wake-on-LAN)](#6-waking-the-ugreen-nas-wake-on-lan)
- [7. The log](#7-the-log)
- [8. Security](#8-security)
- [9. Troubleshooting](#9-troubleshooting)
- [10. License](#10-license)

---

## 1. Requirements

- A **Synology** running DSM 7 with **Container Manager**, and the UPS
  plugged in by USB and recognised (*Control Panel > Hardware & Power >
  UPS*).
- A **UGREEN NAS** running UGOS, on the same local network.
- **A free IP address** on the local network for the relay, outside the
  router's DHCP range.

Nothing to compile or build: the container starts from the official
`python:3.12-alpine` image, which Container Manager downloads by itself, and
the script is included in the compose file.

## 2. The files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The whole project: container, relay script, networks. It is the only file you need. |
| `CHANGELOG.md` | The version history. |
| `LICENSE`, `NOTICE` | The GPL v3 license and its additional terms. |

## 3. Installation

### On the Synology: open the NUT server to the relay

1. *Control Panel > Hardware & Power > UPS*.
2. Tick **Enable network UPS server**.
3. Click **Permitted Synology NAS Devices** and add **172.31.93.2** (the
   relay's address on the internal network).
4. Apply.

### On the Synology: create the project

1. Open **Container Manager > Project > Create**.
2. Project name: `truchement`; path: any folder (for instance
   `/docker/truchement`).
3. Source: **Create docker-compose.yml**, and paste the content of
   [`docker-compose.yml`](docker-compose.yml).
4. Adapt the lines marked « à adapter » (see [4. Settings](#4-settings)): the
   relay's LAN address, the LAN subnet, and the UGREEN NAS address in
   `ALLOWED_CLIENTS`.
5. Confirm: Container Manager downloads the image, creates both networks and
   starts the `truchement` container.

In the container log (*Container Manager > Container > truchement > Log*),
three lines should appear:

```
2026-10-23 14:02:11  Truchement, par ETDEL — https://github.com/theptitprince/Truchement — version 1.1.0
2026-10-23 14:02:11  WoL inactif (WOL_MAC vide)
2026-10-23 14:02:11  Relais NUT 0.0.0.0:3493 -> 172.31.93.1:3493  (UPS ups0 <-> ups, clients 192.168.1.201)
```

The log messages are in French; the ones that matter are explained in
[7. The log](#7-the-log) and [9. Troubleshooting](#9-troubleshooting). To wake
the UGREEN NAS, see [6. Waking the UGREEN NAS](#6-waking-the-ugreen-nas-wake-on-lan).

### On the UGREEN: listen to the relay

In the UGOS UPS settings, choose the **network UPS client (NUT)** mode and
enter the relay's address, **192.168.1.210**. The credentials UGOS asks for
don't matter: the relay replaces them with the Synology's. UGOS should then
show the UPS, its status and its load.

## 4. Settings

### The addresses

| Where | Example value | What it is for |
|---|---|---|
| `networks > lan > ipv4_address` | `192.168.1.210` | Relay's address on the LAN, the one you give to UGOS. **To adapt.** |
| `networks: lan > subnet` (at the bottom) | `192.168.1.0/24` | LAN subnet. **To adapt.** |
| `networks: lan > parent` | `ovs_eth0` | Synology network interface. `eth0` if Open vSwitch is disabled, `ovs_bond0` or `bond0` with link aggregation. |
| `networks > interne > ipv4_address` | `172.31.93.2` | Relay's address on the internal network, to allow in DSM. |
| `networks: interne > gateway` | `172.31.93.1` | The Synology as seen from the container. Change it only if it clashes with another network. |

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `UPSTREAM_HOST` | `192.168.1.200` | NUT server to read. In the compose file: `172.31.93.1`, the Synology as seen from the container. |
| `UPSTREAM_PORT` | `3493` | NUT server port. |
| `LISTEN_HOST` | `0.0.0.0` | Relay listening address. |
| `LISTEN_PORT` | `3493` | Relay listening port. |
| `ALLOWED_CLIENTS` | *(empty = everyone)* | Addresses allowed to connect, comma-separated. **Fill it in**: see [8. Security](#8-security). |
| `CLIENT_UPS` | `ups0` | UPS name on the client side (UGOS). |
| `SERVER_UPS` | `ups` | UPS name on the server side (DSM). |
| `NUT_USER` | `monuser` | Username sent to the NUT server (DSM's is fixed). |
| `NUT_PASS` | `secret` | Password sent to the NUT server (DSM's is fixed). |
| `VERBOSE` | `0` | `0`: short log; `1`: the whole dialogue. |
| `TZ` | `Europe/Paris` | Time zone of the log timestamps. |
| `WOL_MAC`, `WOL_IP`, `WOL_PORT`, `WOL_BROADCAST`, `WOL_POLL`, `WOL_STABLE`, `WOL_RETRY`, `WOL_MAX`, `WOL_BOOT` | *(empty `WOL_MAC` = off)* | Wake-on-LAN of the UGREEN NAS: see [6. Waking the UGREEN NAS](#6-waking-the-ugreen-nas-wake-on-lan). |

After a change: *Container Manager > Project > truchement > Action > Build*
(or Stop then Start) so that the container picks it up.

## 5. How it works

**Why an address of its own.** DSM's NUT server already holds port 3493 on
the Synology's address: the relay cannot sit there too. So it gets its own
LAN address through a **macvlan** Docker network: to UGOS, it is just one
more machine on the network.

**Why an internal network.** A macvlan container cannot reach its own host
through the host's LAN address: that is a limitation of the macvlan driver.
The relay therefore goes through a second network, a small Docker
**bridge** (`172.31.93.0/24`) whose gateway `172.31.93.1` is the Synology
itself. That is how it reads the NUT server, showing up with the address
`172.31.93.2` — hence the permission to grant in DSM.

**What the relay does.** NUT is a text protocol, one command per line.
Truchement reads each line and:

- from UGOS to the Synology, replaces `ups0` with `ups`, and the `USERNAME`
  / `PASSWORD` lines with `monuser` / `secret`;
- from the Synology to UGOS, replaces `ups` with `ups0`.

Only the UPS name as a standalone word is replaced: variable names
(`ups.status`, `ups.load`…) and quoted values stay intact. Everything else
goes through untouched, including the `LOGIN` and `FSD` commands that
trigger the machines' shutdown.

## 6. Waking the UGREEN NAS (Wake-on-LAN)

### The gap

On battery, the UGREEN NAS shuts down first (UGOS shutdown delay), the
Synology a few minutes later (DSM delay: safe mode, then the UPS cuts its
outlets). Both NAS restart by themselves when the power returns, thanks to
their restart-after-power-failure setting.

But if the power comes back **after the UGREEN has shut down and before the
UPS cuts its outlets**, the UGREEN never lost power: its automatic restart
is not triggered, and **it stays off indefinitely**.

```
mains        ━━━━━━━━┓                      ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                     ┗━━━━━ outage ━━━━━━━━━┛
UGREEN NAS   ━━━━━━━━━━━━━━━━━┓ shutdown (UGOS)
                              ┗━━━━━━━━━━━━━━━━━━ off ……… without WoL: forever
Synology     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ (its delay is not reached)
                                            ↑ the gap: Truchement sends the magic packet
```

### What Truchement does

1. **It reads the UPS status itself** from DSM's NUT server, every
   `WOL_POLL` seconds, without waiting for UGOS's questions — once off, the
   UGREEN asks nothing. It opens no session (`LOGIN`): DSM does not have to
   wait for it before shutting down.
2. **It arms** as soon as the UPS goes on battery (`OB`), low battery (`LB`)
   or forced shutdown (`FSD`). An unreachable NUT server (DSM in safe mode)
   does not disarm it.
3. **When the power returns**, once the `OL` status has been stable for
   `WOL_STABLE` seconds, it checks whether the UGREEN answers (TCP connection
   to `WOL_IP:WOL_PORT`). If not, it sends the **magic packet**, then again
   every `WOL_RETRY` seconds, up to `WOL_MAX` attempts. It disarms as soon as
   the UGREEN answers, or gives up after the last attempt.
4. **After a complete power loss**, the container restarts with the
   Synology: if the latter booted less than `WOL_BOOT` seconds ago,
   Truchement arms straight away. Usually the UGREEN restarts by itself and
   Truchement merely notes it; otherwise, it wakes it up.

A UGREEN NAS switched off by hand is **never** switched back on outside
these two cases (going on battery, recent Synology boot).

While the UPS reports a low battery (`OL LB`: power back, battery not yet
recharged), Truchement waits: the wake-up goes out once `LB` clears.

### Requirements

- **Wake-on-LAN enabled in UGOS** (hardware / power settings of the UGREEN
  NAS).
- **Restart after power failure kept on** in UGOS and DSM: it is what
  switches the NAS back on after a complete power loss, because some network
  cards no longer listen to Wake-on-LAN after a total loss of power.
- **At least 5 minutes between** the UGOS shutdown delay (the shorter one)
  and the DSM one, so that the UGREEN is fully off well before DSM enters
  safe mode.
- A **fixed address** for the UGREEN (DHCP reservation) and its **MAC**
  (UGOS network settings).

### Settings

| Variable | Default | Purpose |
|---|---|---|
| `WOL_MAC` | *(empty = off)* | MAC of the UGREEN NAS, `AA:BB:CC:DD:EE:FF` (dashes or no separator accepted). **To adapt.** |
| `WOL_IP` | *(the only IP in `ALLOWED_CLIENTS`)* | Address of the UGREEN NAS. Required if `ALLOWED_CLIENTS` does not hold exactly one address. |
| `WOL_PORT` | `445` | TCP port proving the UGREEN is on (445 = SMB file sharing). If SMB is off, pick another open port, the UGOS web interface's for instance. |
| `WOL_BROADCAST` | `192.168.1.255` | LAN broadcast address (`192.168.1.255` for `192.168.1.0/24`). **To adapt.** |
| `WOL_POLL` | `10` | Seconds between two UPS status reads. |
| `WOL_STABLE` | `120` | Seconds of stable mains power before the first wake-up. |
| `WOL_RETRY` | `60` | Seconds between two attempts. |
| `WOL_MAX` | `15` | Attempts before giving up. |
| `WOL_BOOT` | `900` | Arm at start-up if the Synology booted less than `WOL_BOOT` seconds ago (`0` = never). |

The magic packet (6 bytes `FF` then the MAC 16 times) goes out over UDP to
`WOL_BROADCAST`, ports 9 and 7, three times per attempt. It needs the **LAN
broadcast**, not `255.255.255.255`: the container has two networks, and only
the LAN broadcast is sure to leave through the macvlan interface. At
start-up, Truchement checks which way the packet would leave and warns if it
is not the interface that reaches the UGREEN.

### In the log

At start-up:

```
2026-10-26 08:00:05  WoL actif : MAC 6C:1F:F7:12:34:56, cible 192.168.1.201:445, broadcast 192.168.1.255  (sondage 10 s, stable 120 s, relance 60 s, 15 essais, armé au lancement : si hôte démarré depuis < 900 s)
```

The gap, filled (`armé` = armed, `paquet magique envoyé` = magic packet
sent, `joignable, désarmé` = reachable, disarmed):

```
2026-10-26 03:12:40  *** ups.status : OL -> OB DISCHRG
2026-10-26 03:12:40  *** WoL : armé (passage sur batterie)
2026-10-26 03:21:15  *** ups.status : OB DISCHRG -> OL CHRG
2026-10-26 03:23:16  *** WoL : paquet magique envoyé à 6C:1F:F7:12:34:56 (1/15)
2026-10-26 03:24:17  *** WoL : paquet magique envoyé à 6C:1F:F7:12:34:56 (2/15)
2026-10-26 03:25:20  *** WoL : 192.168.1.201 joignable, désarmé
```

After a complete power loss, the UGREEN having restarted by itself
(`démarrage récent du Synology` = recent Synology boot):

```
2026-10-26 08:00:05  *** WoL : armé (démarrage récent du Synology)
2026-10-26 08:00:05  *** ups.status : ? -> OL
2026-10-26 08:02:06  *** WoL : 192.168.1.201 joignable, désarmé
```

No answer from the UGREEN after the last attempt (`abandon après 15 essais` =
giving up after 15 attempts):

```
2026-10-26 03:38:20  !!! WoL : abandon après 15 essais
```

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `!!! WoL : adresse MAC invalide …`, `WOL_IP obligatoire …`, `… hors limites` | Wrong setting: the wake-up is disabled, the relay keeps working normally. Fix the variable and restart the container. |
| `!!! WoL : le broadcast part de 172.31.93.2 mais la cible est jointe depuis 192.168.1.210` | Wrong `WOL_BROADCAST`: use the LAN broadcast address (`192.168.1.255` for `192.168.1.0/24`). |
| Packets sent, the UGREEN stays off, then `!!! WoL : abandon après 15 essais` | Wake-on-LAN not enabled in UGOS, wrong MAC, or wrong broadcast (another subnet). If the UGREEN did come on despite the give-up: `WOL_PORT` is closed on the UGREEN, pick an open port. |
| The UGREEN does not wake after a complete power loss, but does wake in the gap | Its network card does not listen to Wake-on-LAN after a total loss of power: keep restart after power failure on in UGOS, that is what covers this case. |
| `*** WoL : … joignable, désarmé` while the UGREEN is off | `WOL_IP` points to another machine: check the UGREEN's address. |
| `!!! WoL : uptime de l'hôte illisible` | No arming at start-up; the rest of the wake-up works. |

## 7. The log

By default (`VERBOSE: "0"`), the log stays short. It records:

- the origin notice, the version and the configuration at start-up;
- each authentication (`USERNAME`, masked `PASSWORD`, `LOGIN`);
- NUT errors (except `FEATURE-NOT-CONFIGURED`, which is irrelevant);
- **each UPS status change**, for instance
  `*** ups.status : OL -> OB DISCHRG` when the power fails;
- refused connections (`connexion refusée`) and an unreachable NUT server
  (`injoignable`);
- the Wake-on-LAN events (`*** WoL : …`, `!!! WoL : …`), see
  [6. Waking the UGREEN NAS](#6-waking-the-ugreen-nas-wake-on-lan).

With `VERBOSE: "1"`, the whole dialogue is shown, except the content of
variable lists (only their line count is noted). Handy for debugging, too
chatty for everyday use: UGOS polls the UPS every few seconds.

## 8. Security

The relay **accepts any credentials** from an allowed client and replaces
them with DSM's: `ALLOWED_CLIENTS` is therefore the only barrier. Fill it
with the UGREEN NAS address (and, temporarily, a test PC's), and give the
UGREEN NAS a fixed address (DHCP reservation).

The `monuser` / `secret` credentials are not a secret: they are DSM's NUT
server's fixed, public ones. What protects that server is its list of
permitted devices — which only holds the relay.

## 9. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `!!! connexion refusée de 192.168.1.x (hors ALLOWED_CLIENTS)` | The address is not in `ALLOWED_CLIENTS`. Add it, then restart the container. |
| `!!! serveur NUT 172.31.93.1:3493 injoignable` | Network UPS server not enabled in DSM, or UPS not recognised by the Synology. |
| `ERR ACCESS-DENIED` on `LOGIN` | `172.31.93.2` is not in DSM's *Permitted Synology NAS Devices*. |
| The project won't start, error on the `lan` network | Wrong `parent` interface (`ovs_eth0`, `eth0`, `ovs_bond0`…), or address already taken on the LAN. The exact name can be read over SSH with `ip link`. |
| Error on the `interne` network (*pool overlaps*) | `172.31.93.0/24` is already used by another Docker network: pick another subnet, and update the permitted address in DSM. |
| The Synology does not answer pings to 192.168.1.210 | Expected: that is the macvlan limitation described above. Test from another machine. |
| UGOS does not see the UPS | Set `VERBOSE: "1"` and look at what UGOS asks and what DSM answers. |
| The UGREEN stays off after a power failure | See the wake-up troubleshooting at the end of [6. Waking the UGREEN NAS](#6-waking-the-ugreen-nas-wake-on-lan). |

To test from a PC with the NUT tools installed, add its address to
`ALLOWED_CLIENTS`, then:

```bash
upsc ups0@192.168.1.210
```

The command should list the UPS variables (`ups.status: OL`,
`battery.charge`…).

## 10. License

Truchement is **free software**, released under the
**GNU General Public License, version 3 or later** (GPL-3.0-or-later).

Copyright © 2026 ETDEL.

In practice, you may use the program freely, including professionally,
study it, modify it and redistribute it — provided that any redistributed
version, modified or not, stays under the same license, with its source code
and the copyright notice. The program comes **without any warranty**: it
relays the UPS status, it does not replace checking your machines' shutdown
chain.

### Additional terms (section 7)

The GPL v3 lets the author add a few terms (section 7). Truchement's are in the
[`NOTICE`](NOTICE) file, in French and English:

- **Attribution**: every copy or modified version must keep, visible to the user and in its
  documentation, the notice:

  > Truchement, par ETDEL — https://github.com/theptitprince/Truchement

  A modified version may prefix it with "D'après" (or "Based on") and add its own credits,
  without removing or hiding it.
- **Origin**: a modified version must state clearly that it has been modified, and must not
  present itself as the original, nor as endorsed by ETDEL.

In Truchement, this notice is written to the container log at every start-up.

The full license text is in the repository's `LICENSE` file and at
<https://www.gnu.org/licenses/gpl-3.0.html>. The `docker-compose.yml` file
carries a header pointing to `LICENSE` and `NOTICE`.

---

*Truchement — ETDEL 2026 — GPL-3.0-or-later*
