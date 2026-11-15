# Home Assistant Local HTTPS Setup (Debian Docker + UI)

## 1. Generate Self-Signed Certificates on Debian

Run the following commands on your Debian host machine (replace `/home/arduino/homeassistant_config` and `192.168.50.69` if your config path or server IP is different).

```bash
# 1. Create the SSL directory inside your mapped Home Assistant config folder
mkdir -p /home/arduino/homeassistant_config/ssl
cd /home/arduino/homeassistant_config/ssl

# 2. Generate the certificate (fullchain.pem) and private key (privkey.pem)
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout privkey.pem \
  -out fullchain.pem \
  -days 7300 \
  -subj "/CN=homeassistant.local" \
  -addext "subjectAltName=DNS:homeassistant.local,IP:127.0.0.1,IP:192.168.50.69,IP:192.168.50.70" \
  -addext "extendedKeyUsage=serverAuth"

# 3. Set secure file permissions
chmod 644 fullchain.pem
chmod 600 privkey.pem
```

> [!NOTE]
> Because your mapped config volume mounts to `/config` inside the Home Assistant Docker container, these files will be located at `/config/ssl/fullchain.pem` and `/config/ssl/privkey.pem` inside Home Assistant.

## 2. Configure SSL in the Home Assistant UI

1. Open your Home Assistant web interface and go to **Settings > System > Network**.
2. Scroll down to the **HTTP Server** (or **SSL/TLS**) section.
3. Enter the container paths to your certificate and key:
   - **SSL certificate path:** `/config/ssl/fullchain.pem`
   - **SSL key path:** `/config/ssl/privkey.pem`
   - **SSL profile:** `Intermediate v6` (default/recommended)
   - **Server port:** `8123` (or change to `443` if you want to omit the port from your URL)
4. Click **Save**.

## 3. Update Home Assistant Local URL (Recommended)

While still on the **Settings > System > Network** page:

1. Locate the **Home Assistant URL** section.
2. Under **Local network**, disable **Automatic** if enabled.
3. Enter your new HTTPS URL (e.g., `https://192.168.50.69:8123` or `https://homeassistant.local:8123`).
4. Click **Save**.

## 4. Restart Home Assistant

1. Go to **Settings > System**.
2. Click the **Power icon** (top-right corner) > **Restart Home Assistant** > **Restart**.
3. Once restarted, access Home Assistant via `https://192.168.50.69:8123` (accept the browser's self-signed certificate warning to proceed).
