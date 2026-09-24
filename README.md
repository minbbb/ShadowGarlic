# ShadowGarlic

**English** · [Русский](README.ru.md)

Two docker compose environments: **server** (i2pd + ssserver-rust) and **client**
(i2pd + sslocal-rust). Traffic from the regular internet goes through
Shadowsocks, wrapped into I2P streams between the server and client i2pd tunnels.

```
           CLIENT                                     SERVER
┌───────────────────────────┐               ┌────────────────────────┐
│  Application on the host  │               │ To the public internet │
│            │              │               │            ▲           │
│            ▼              │               │            │           │
│      sslocal-rust         │               │            │           │
│   SOCKS5 127.0.0.1:1080   │               │   ssserver-rust:8388   │
│            │              │               │            ▲           │
│            ▼              │               │            │           │
│           i2pd            │      I2P      │          i2pd          │
│    client tunnel :7655 ─────────────────────> server tunnel :8388  │
└───────────────────────────┘               └────────────────────────┘
```

- The client i2pd builds a tunnel to the server's **b32 address** and listens
  on TCP port `7655`; sslocal connects to this port and serves a SOCKS5 proxy
  to the host.
- sslocal encrypts the traffic, i2pd delivers it over I2P to the server tunnel,
  the i2pd on the server forwards it to ssserver, which decrypts it and reaches
  the internet.

## Table of contents

1. [Server setup](#1-server-setup)
2. [Client setup](#2-client-setup)
3. [Verifying it works](#3-verifying-it-works)
4. [Limitations and notes](#limitations-and-notes)

## 1. Server setup

```bash
git clone https://github.com/minbbb/ShadowGarlic
cd ShadowGarlic
cp .env.example .env
```

Edit `.env`:

| Variable              | Default                    | Purpose                                                                                              |
| --------------------- | -------------------------- | ---------------------------------------------------------------------------------------------------- |
| `SS_METHOD`           | `aes-256-gcm`              | Shadowsocks cipher                                                                                   |
| `SS_PASSWORD`         | `ChangeMe_strong_password` | Shadowsocks password - **you must change it**                                                        |
| `SS_SERVER_PORT`      | `8388`                     | ssserver port inside the compose network. Must match `port = 8388` in `config/tunnels.conf.template` |
| `SS_LOCAL_PORT`       | `1080`                     | SOCKS5 port on the client (host)                                                                     |
| `I2PD_CONSOLE_PASS`   | `ChangeMe_strong_password` | i2pd webconsole password (Basic Auth) - **you must change it**                                       |
| `I2P_INBOUND_LENGTH`  | `3`                        | Inbound I2P tunnel hops (0-8) - see section below                                                    |
| `I2P_OUTBOUND_LENGTH` | `3`                        | Outbound I2P tunnel hops (0-8) - see section below                                                   |
| `I2PD_IMAGE_TAG`      | `release-2.61.0`           | i2pd image tag                                                                                       |
| `SS_IMAGE_TAG`        | `v1.25.0`                  | shadowsocks-rust image tag                                                                           |

> IMPORTANT about passwords (`SS_PASSWORD` and `I2PD_CONSOLE_PASS`): avoid the
> characters `$`, `"`, `\` - they break compose interpolation (the values end up
> in the CLI args `-k ...` and `--http.pass="..."`).

Run:

```bash
docker compose up -d
```

Check that they started:

```bash
docker compose ps
```

### Getting the server tunnel's b32 address (needed by the client)

The first start creates the `shadowsocks.dat` keys and, after building the lease
set, logs the address:

```bash
docker compose logs i2pd | grep "Local address"
# Example output:
# Clients: Local address bg3............m.b32.i2p loaded
```

Alternatively, open the webconsole by forwarding the port over SSH
(`ssh -p 22 -L 7070:127.0.0.1:7070 root@<server_IP>` → on your PC open
`http://localhost:7070`, login `i2pd`, password - the `I2PD_CONSOLE_PASS`
value from `.env`) → the **I2P tunnels** page → the **Server tunnels** section:
`<b32>.b32.i2p` is there too.

> The webconsole is not directly reachable at `http://<server_IP>:7070`: docker
> compose publishes the port only on `127.0.0.1`, and the console itself is
> protected by Basic Auth.

### Configuring the number of hops (I2P)

The length of the I2P tunnels is set by two `.env` variables:

| Variable              | Range | Default | What it sets         |
| --------------------- | ----- | ------- | -------------------- |
| `I2P_INBOUND_LENGTH`  | 0-8   | `3`     | Inbound tunnel hops  |
| `I2P_OUTBOUND_LENGTH` | 0-8   | `3`     | Outbound tunnel hops |

- Fewer hops → lower latency and higher speed, but weaker anonymity.
- How it is configured technically: the values from `.env` are passed into the
  container via `environment:`, and the i2pd container entrypoint substitutes
  them into `inbound.length`/`outbound.length`:
    - server - by rendering the `config/tunnels.conf.template` template into
      `/home/i2pd/data/tunnels.conf`;
    - client - by injecting them into your `client/tunnels.conf` (the file with
      the b32 address).
- To apply after editing `.env`: `docker compose up -d` (changing the value
  recreates the i2pd container, which rebuilds its tunnels with the new length).
- You can check the actual length of individual tunnels in the i2pd log
  (`docker compose logs i2pd | grep "Parameters for tunnel"`): while building,
  i2pd prints a line like `5 inbound (3 hops), 5 outbound (4 hops)`.

### Address reset (removing the volume)

The server tunnel address (`shadowsocks.dat`, the b32 address), netdb and the
rest of the i2pd data are stored in the named volume `shadowgarlic_i2pd-data`.
To reset them and get a **new b32 address**:

```bash
# on the server:
docker compose down -v                   # stops the containers and removes the volume
docker volume rm shadowgarlic_i2pd-data  # just removes the volume
```

> ⚠️ After the reset the address changes - update `destination` in
> `client/tunnels.conf` on the client (see step 2), otherwise the client tunnel
> will not connect to the server. The client volume is reset the same way:
> `docker compose -f docker-compose.client.yml down -v`.

## 2. Client setup

On the client machine:

```bash
# 1. Create .env from the template:
cp .env.example .env
```

Alternatively, copy `.env` from the server.

> `SS_PASSWORD` and `SS_METHOD` on the client **must match** the server ones.
> `I2PD_CONSOLE_PASS` and `SS_LOCAL_PORT` are optional (the client console is
> not published to the host, and the SOCKS5 port is set by this file).

```bash
# 2. Copy the tunnel config from the template:
cp client/tunnels.conf.example client/tunnels.conf
```

Open `client/tunnels.conf` and paste the b32 address from the step above:

```
destination = <b32-address>.b32.i2p
```

Run:

```bash
docker compose -f docker-compose.client.yml up -d
```

### Using from a phone

Alternatively, the proxy can be used from a phone: set up the tunnel in an I2P
app using the server's b32 address and connect to it with any
Shadowsocks-capable client using `SS_PASSWORD`/`SS_METHOD` from `.env`. In the
I2P app point the tunnel to the server's `destinationport` (`8388` by default)
and specify a local port (e.g. `7655`) - the Shadowsocks client then connects to
`127.0.0.1:7655`.

## 3. Verifying it works

On the client:

```bash
curl -x socks5h://127.0.0.1:1080 https://api.ipify.org
```

Returns the **server's public IP**.

> Note the `socks5h` (the domain is resolved through the proxy - server-side).
> With `socks5` (no `h`) the domain is resolved locally by the client's system
> resolver and will not go through the tunnel.

On the server, check the tunnel activity via the webconsole
(`http://localhost:7070` after the SSH forward, see above → I2P tunnels).

View the logs:

```bash
docker compose logs -f i2pd                                # server i2pd logs
docker compose -f docker-compose.client.yml logs -f i2pd   # client i2pd logs
```

## Limitations and notes

- **The client SOCKS5 (port `SS_LOCAL_PORT`) is published on `0.0.0.0`** - the
  proxy without authentication is available to all devices on the host's local
  network. If access is needed only from the machine itself, change it to
  `127.0.0.1:${SS_LOCAL_PORT}:${SS_LOCAL_PORT}` in `docker-compose.client.yml`.
- **The i2pd webconsole is protected by Basic Auth** (password
  `I2PD_CONSOLE_PASS` from `.env`); on the server port 7070 is bound to
  `127.0.0.1`, access is via SSH forwarding.
