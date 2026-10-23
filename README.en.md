<div align="center">

# Truchement — Synology → UGOS NUT relay

**One UPS on the Synology, the UGREEN warned too.**

![Version](https://img.shields.io/badge/version-1.0-2ea44f)
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

```
 UGREEN (UGOS)                    TRUCHEMENT                       SYNOLOGY (DSM)
 NUT client        ----------->   192.168.1.210:3493   ---------->  NUT server
 "ups0"                           (macvlan, LAN)                    "ups", USB
 UGOS credentials                 172.31.93.2  ── internal net ──>  172.31.93.1:3493
                   <-----------   ups ↔ ups0           <----------  monuser / secret
```

Current version: 1.0 — the release history is in
[CHANGELOG.md](CHANGELOG.md) (in French).

## Contents

- [1. Requirements](#1-requirements)
- [2. The files](#2-the-files)
- [3. Installation](#3-installation)
- [4. Settings](#4-settings)
- [5. How it works](#5-how-it-works)
- [6. The log](#6-the-log)
- [7. Security](#7-security)
- [8. Troubleshooting](#8-troubleshooting)
- [9. License](#9-license)

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
two lines should appear:

```
2026-10-23 14:02:11  Truchement, par ETDEL — https://github.com/theptitprince/Truchement — version 1.0
2026-10-23 14:02:11  Relais NUT 0.0.0.0:3493 -> 172.31.93.1:3493  (UPS ups0 <-> ups, clients 192.168.1.201)
```

The log messages are in French; the ones that matter are explained in
[6. The log](#6-the-log) and [8. Troubleshooting](#8-troubleshooting).

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
| `ALLOWED_CLIENTS` | *(empty = everyone)* | Addresses allowed to connect, comma-separated. **Fill it in**: see [7. Security](#7-security). |
| `CLIENT_UPS` | `ups0` | UPS name on the client side (UGOS). |
| `SERVER_UPS` | `ups` | UPS name on the server side (DSM). |
| `NUT_USER` | `monuser` | Username sent to the NUT server (DSM's is fixed). |
| `NUT_PASS` | `secret` | Password sent to the NUT server (DSM's is fixed). |
| `VERBOSE` | `0` | `0`: short log; `1`: the whole dialogue. |
| `TZ` | `Europe/Paris` | Time zone of the log timestamps. |

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

## 6. The log

By default (`VERBOSE: "0"`), the log stays short. It records:

- the origin notice, the version and the configuration at start-up;
- each authentication (`USERNAME`, masked `PASSWORD`, `LOGIN`);
- NUT errors (except `FEATURE-NOT-CONFIGURED`, which is irrelevant);
- **each UPS status change**, for instance
  `*** ups.status : OL -> OB DISCHRG` when the power fails;
- refused connections (`connexion refusée`) and an unreachable NUT server
  (`injoignable`).

With `VERBOSE: "1"`, the whole dialogue is shown, except the content of
variable lists (only their line count is noted). Handy for debugging, too
chatty for everyday use: UGOS polls the UPS every few seconds.

## 7. Security

The relay **accepts any credentials** from an allowed client and replaces
them with DSM's: `ALLOWED_CLIENTS` is therefore the only barrier. Fill it
with the UGREEN NAS address (and, temporarily, a test PC's), and give the
UGREEN NAS a fixed address (DHCP reservation).

The `monuser` / `secret` credentials are not a secret: they are DSM's NUT
server's fixed, public ones. What protects that server is its list of
permitted devices — which only holds the relay.

## 8. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `!!! connexion refusée de 192.168.1.x (hors ALLOWED_CLIENTS)` | The address is not in `ALLOWED_CLIENTS`. Add it, then restart the container. |
| `!!! serveur NUT 172.31.93.1:3493 injoignable` | Network UPS server not enabled in DSM, or UPS not recognised by the Synology. |
| `ERR ACCESS-DENIED` on `LOGIN` | `172.31.93.2` is not in DSM's *Permitted Synology NAS Devices*. |
| The project won't start, error on the `lan` network | Wrong `parent` interface (`ovs_eth0`, `eth0`, `ovs_bond0`…), or address already taken on the LAN. The exact name can be read over SSH with `ip link`. |
| Error on the `interne` network (*pool overlaps*) | `172.31.93.0/24` is already used by another Docker network: pick another subnet, and update the permitted address in DSM. |
| The Synology does not answer pings to 192.168.1.210 | Expected: that is the macvlan limitation described above. Test from another machine. |
| UGOS does not see the UPS | Set `VERBOSE: "1"` and look at what UGOS asks and what DSM answers. |

To test from a PC with the NUT tools installed, add its address to
`ALLOWED_CLIENTS`, then:

```bash
upsc ups0@192.168.1.210
```

The command should list the UPS variables (`ups.status: OL`,
`battery.charge`…).

## 9. License

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
