# Uno Q Home Assistant Setup

Welcome! Here's a walkthrough of how I currently run Home Assistant, Matter, and MQTT on my Arduino Uno Q. I hope it helps you get yours up and running too.

## 1. Increase `/root` Size

The Uno Q's default `/root` partition doesn't have enough space for large Docker images. Reflash it with a larger root partition:

```powershell
.\arduino-flasher-cli.exe flash unoq --root-size=16GB
```
## 2. Make required directories for HA, Matter and MQTT

```bash
mkdir -p ~/homeassistant_config
mkdir -p ~/matter_server_data
mkdir -p ~/mosquitto
```


## 3. Create the Docker Compose Config

Create `~/compose.yaml` with the following configuration:

- Update `TZ` to your [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) (example: `America/Santo_Domingo`).
- Update `/home/arduino/...` volume paths if your username is not `arduino`.
- Set `OTBR_INTERFACE` to your active network interface (example: `wlp3s0`). Run `ip -br link` to list available interfaces.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    privileged: true
    restart: unless-stopped
    network_mode: host
    environment:
      - TZ=America/Santo_Domingo
    volumes:
      - /home/arduino/homeassistant_config:/config
      - /run/dbus:/run/dbus:ro

  matter-server:
    container_name: matter-server
    image: ghcr.io/home-assistant-libs/python-matter-server:stable
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

  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - /home/arduino/mosquitto/config:/mosquitto/config
      - /home/arduino/mosquitto/data:/mosquitto/data
      - /home/arduino/mosquitto/log:/mosquitto/log
```

Start the containers:

```bash
docker compose up -d
```

## 4. Securing Your HA, MQTT, and Matter

Please see:

- [Home Assistant Local HTTPS Setup (Debian Docker + UI)](./HA%20SSL.md)
- [Home-Assistant-Arduino-Q-MQTT-Example](https://github.com/rcpeters/Home-Assistant-Arduino-Q-MQTT-Example/blob/main/SECURING.md)
- [Matter Setup](./HA%20Matter%20Setup.md)

## 5. Migrate to a New Machine

See [Migrate Home Assistant.md](./Migrate%20Home%20Assistant.md).
