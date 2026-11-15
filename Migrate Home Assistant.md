# Migrate Home Assistant to a New Machine

If you are migrating to a new machine, configure it to use the old machine's IP address.

1. On the **old machine**, temporarily allow passwordless `sudo` for `rsync` (since `root` owns the data directories):

   ```bash
   sudo visudo
   ```

   Add this line at the bottom:

   ```text
   arduino ALL=(ALL) NOPASSWD: /usr/bin/rsync
   ```

2. On the **new machine**, copy the data directories from the old machine (replace `arduino` and `192.168.50.69` with your old machine's username and IP):

   ```bash
   sudo rsync -avz --rsync-path="sudo rsync" arduino@192.168.50.69:~/homeassistant_config
   sudo rsync -avz --rsync-path="sudo rsync" arduino@192.168.50.69:~/matter_server_data
   sudo rsync -avz --rsync-path="sudo rsync" arduino@192.168.50.69:~/mosquitto
   sudo rsync -avz --rsync-path="sudo rsync" arduino@192.168.50.69:~/pki
   ```

3. On the **new machine**, start the containers and verify they are running:

   ```bash
   docker compose up -d
   docker compose ps
   ```

   Check the logs if anything looks off, and confirm Home Assistant loads at `http://<machine-ip>:8123`:

   ```bash
   docker compose logs -f
   ```

