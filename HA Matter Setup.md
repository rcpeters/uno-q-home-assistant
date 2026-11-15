# Setting Up Home Assistant & Python Matter Server on Arduino Q

This guide walks you through setting up a local smart home server on the **Arduino Uno Q / Ventuno Q** running **Home Assistant** and the official **Python Matter Server** (`python-matter-server`) inside Docker, building upon the official [Smart Home Hub with VENTUNO Q and Home Assistant](https://docs.arduino.cc/tutorials/ventuno-q/home-assistant/) tutorial.

## Prerequisites

- **Hardware**: Arduino Uno Q / Ventuno Q connected to your local network (Wi-Fi or Ethernet)
- **Power**: Compatible USB-C power supply (7–24V, $\ge 3$A recommended)
- **Terminal Access**: SSH access to the board (`ssh arduino@<board-ip>`)
- **Docker**: Pre-installed on the Arduino Linux environment

## 1. Install mDNS Tools & Verify IPv6 on the Host

Matter uses **mDNS (Multicast DNS)** for device discovery and **IPv6** for operational communication. Additionally, the `circuitmatter` library used by this project relies on `avahi-utils` (`avahi-publish-service`) to advertise the Arduino Q as a commissionable Matter node.

SSH into your board and install `avahi-daemon` and `avahi-utils`:

```bash
sudo apt-get update
sudo apt-get install -y avahi-daemon avahi-utils
sudo systemctl enable --now avahi-daemon
```

> [!TIP]
> Restarting your user session (logging out and logging back in via SSH) is likely needed for the newly installed services, D-Bus session bus, and permissions to take full effect.

Verify that IPv6 is enabled on your active network interface (look for an `inet6` link-local `fe80::...` address):

```bash
ip -6 addr show
```

> [!IMPORTANT]
> Matter requires IPv6 link-local addresses (`fe80::/10`) even on an IPv4-only ISP connection. Do not disable IPv6 in `/etc/sysctl.conf` on the board.

## 2. Create Persistent Directories

Creating dedicated directories on the host ensures your Home Assistant configurations, Matter fabric credentials, and commissioned node certificates persist across container updates and reboots.

SSH into your board and run:

```bash
# Create Home Assistant configuration directory
mkdir -p ~/homeassistant_config

# Create Python Matter Server persistent storage directory
mkdir -p ~/matter_server_data
```

## 3. Launch Containers via Docker

You can launch both services using either individual `docker run` commands or a single `compose.yaml` file.

> [!NOTE]
> Both containers must run with `--network=host` (`network_mode: host`) so the Matter Server can send and receive multicast mDNS (`UDP 5353`) and IPv6 UDP (`UDP 5540`) packets directly on your local network interface.

### Option A: Using Docker Run Commands

#### 1. Start the Python Matter Server

```bash
docker run -d \
  --name matter-server \
  --restart unless-stopped \
  --security-opt apparmor=unconfined \
  --network=host \
  -v /home/arduino/matter_server_data:/data \
  -v /run/dbus:/run/dbus:ro \
  ghcr.io/home-assistant-libs/python-matter-server:stable \
  --storage-path /data \
  --paa-root-cert-dir /data/credentials
```

#### 2. Start the Home Assistant Container

Replace `America/Santo_Domingo` with your [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) if different:

```bash
docker run -d \
  --name homeassistant \
  --restart unless-stopped \
  --privileged \
  -e TZ=America/Santo_Domingo \
  -v /home/arduino/homeassistant_config:/config \
  -v /run/dbus:/run/dbus:ro \
  --network=host \
  ghcr.io/home-assistant/home-assistant:stable
```

### Option B: Using Docker Compose (Recommended)

Create `~/compose.yaml` with the following configuration:

- Update `TZ` to your [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) (example: `America/Santo_Domingo`).
- Update `/home/arduino/...` volume paths if your username is not `arduino`.
- Set `OTBR_INTERFACE` to your active network interface (example: `wlp3s0`). Run `ip -br link` to list available interfaces.

```yaml
services:
  matter-server:
    image: ghcr.io/home-assistant-libs/python-matter-server:stable
    container_name: matter-server
    restart: unless-stopped
    network_mode: host
    security_opt:
      - apparmor=unconfined
    environment:
      - OTBR_INTERFACE=wlp3s0
    volumes:
      - /home/arduino/matter_server_data:/data
      - /run/dbus:/run/dbus:ro
    command:
      - --storage-path
      - /data
      - --paa-root-cert-dir
      - /data/credentials

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      - TZ=America/Santo_Domingo
    volumes:
      - /home/arduino/homeassistant_config:/config
      - /run/dbus:/run/dbus:ro
    depends_on:
      - matter-server
```

Start both containers:

```bash
docker compose up -d
```

Verify both containers are running:

```bash
docker ps
```

## 4. Complete Home Assistant Onboarding

1. Open your browser on a computer connected to the same network.
2. Navigate to:
   ```text
   http://<BOARD_IP>:8123
   ```
   *(e.g., `http://192.168.50.69:8123`)*
3. Follow the on-screen wizard to create your admin username and password.

## 5. Add the Matter Integration in Home Assistant

1. In the Home Assistant sidebar, go to **Settings > Devices & Services**.
2. Click **+ Add Integration** in the bottom right corner.
3. Search for **Matter** and select **Matter (BETA)**.
4. When prompted for the connection method:
   - Uncheck *Use the official Home Assistant Matter Server add-on* (if shown).
   - Enter the WebSocket URL for your local container:
     ```text
     ws://localhost:5580/ws
     ```
5. Click **Submit**. Home Assistant is now connected to the Python Matter Server.

## 6. Next Step: Understand Matter Commissioning, Security & Networking

Unlike MQTT—which starts unencrypted and anonymous until you manually configure password files, ACLs, and an OpenSSL CA—Matter enforces end-to-end cryptographic authentication and encryption from the moment a device is commissioned.

Continue with **[Matter Commissioning, Security & Networking](./Commissioning%20and%20Security.md)** to:

- Understand how Matter **PASE**, **CASE**, **Fabrics**, and **Device Attestation Certificates (DACs)** secure every device automatically.
- Share your Arduino Q or ESP32 Matter devices across multiple ecosystems (**Multi-Admin** with Apple Home, Google Home, and Home Assistant).
- Reset or decommission devices cleanly (`matter-device-state.json` and NVS).
- Connect standalone ESP32 devices over Matter-over-Wi-Fi with our **[Example ESP32 Matter (Milieu Panel)](https://github.com/rcpeters/Milieu-Panel)**.
