# Linux presence detection for Home Assistant

This script sets up a local background service to notify Home Assistant immediately when your Linux desktop locks or unlocks.
This allows you to use the state of your desktop for automations.

**NOTE**: This script is **100% vibe coded**, use at your own risk.

## Preparations (Home Assistant Setup)

Before installing the script on your Linux machine, you need to configure Home Assistant to receive the Webhook and store the presence state.

### 1. Generate a Secure Webhook ID
Your Webhook ID acts as the password for your device to push data. It should be unique and hard to guess. You can easily generate a random 24-character string in your Linux terminal using `openssl`:

```bash
openssl rand -hex 12
```
*Save the output string. You will need it for the Home Assistant automation and the setup script.*

### 2. Create the Presence Helper
Since webhooks cannot create entities on their own, we need a virtual switch to hold the presence state (`on` = Present, `off` = Away).
1. In Home Assistant, navigate to **Settings** > **Devices & Services** > **Helpers**.
2. Click **+ Create Helper** and select **Toggle** (Input Boolean).
3. Name it **Desktop Presence** (this creates `input_boolean.desktop_presence`).
4. Click **Create**.

### 3. Create the Webhook Automation
Now, create an automation that listens for the incoming Webhook and updates your presence.
1. Navigate to **Settings** > **Automations & Scenes**.
2. Click **+ Create Automation** > **Create new automation**.
3. Click the three dots (`⋮`) in the top right corner and select **Edit in YAML**.
4. Paste the following configuration, replacing the `webhook_id` with the 24-character string you generated in Step 1:

```yaml
alias: Desktop Presence Webhook Listener
description: Updates the desktop presence state from the Linux script
trigger:
  - platform: webhook
    allowed_methods:
      - POST
    local_only: true
    webhook_id: "YOUR_24_CHARACTER_STRING_HERE"
action:
  # 1. Update the state directly from the script payload ('on' = Present, 'off' = Away)
  - service: input_boolean.turn_{{ trigger.json.state }}
    target:
      entity_id: input_boolean.desktop_presence
  # 2. Start a 10-minute timeout timer
  - delay: "00:10:00"
  # 3. If no new webhooks are received within 10 minutes, force the state to 'off' (AWAY)
  - service: input_boolean.turn_off
    target:
      entity_id: input_boolean.desktop_presence
# Setting mode to 'restart' ensures the 10-minute delay resets every time the script pushes an update
mode: restart
```
*Note: If your Linux machine accesses Home Assistant via an external URL (e.g., Nabu Casa or a reverse proxy), remove `local_only: true` from the YAML.*

## Installation

The included setup script installs the necessary files locally to your user directory without requiring `sudo` privileges.

1. Make the setup script executable:
   ```bash
   chmod +x linux-presence-installer
   ```
2. Run the installer:
   ```bash
   ./linux-presence-installer
   ```
3. Follow the interactive prompts to enter your Home Assistant URL (e.g., `http://192.168.1.50:8123`) and your secure Webhook ID. The script will test the connection and enable the background service.

## Files & Paths

The installer places the following files in your user directory:
* **Executable Script:** `~/.local/bin/ha-lock-listener.sh`
* **Configuration File:** `~/.config/ha-lock/env` (Contains your URL and Webhook ID)
* **Systemd Service:** `~/.config/systemd/user/ha-lock.service`

## Troubleshooting

The service runs entirely in the background and logs all HTTP responses. If states aren't syncing, you can check the real-time logs and HTTP dumps using `journalctl`:

```bash
journalctl --user -u ha-lock.service -f
```

## Uninstallation

To cleanly stop the service, disable it, and remove all files and configurations created by the script, simply run:

```bash
./linux-presence-installer --uninstall
```
