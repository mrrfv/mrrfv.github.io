---
title: Use the Brave Browser Flatpak with the KeePassXC browser extension
date: 2026-05-17 15:00:00 +0100
categories: [Linux]
tags: [linux, security]
---

Flatpak isolates applications by design. While great for security, this isolation breaks native messaging between applications. If you run both Brave and KeePassXC as Flatpaks, the browser extension cannot communicate with the password manager because they exist in their own sandboxes.

To fix this, you must puncture Brave's sandbox, map KeePassXC's runtime into it, and manually define the native messaging host to handle the connection. This is easier than it sounds.

## 1. Update the Brave Flatpak's permissions

Brave needs read access to KeePassXC’s flatpak directories and permission to interact with its proxy socket. Run this to override Brave's permissions:

```bash
flatpak override --user \
  --filesystem={/var/lib,xdg-data}/flatpak/{app/org.keepassxc.KeePassXC,runtime/org.kde.Platform}:ro \
  --filesystem=xdg-run/app/org.keepassxc.KeePassXC:create \
  com.brave.Browser
```

## 2. Create a wrapper script

The Brave flatpak doesn't have `keepassxc-proxy` in its environment, so it requires a wrapper script.

Create the target directory and file:

```bash
mkdir -p ~/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/
nano ~/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/keepassxc-proxy-wrapper.sh
```

Paste the following. You must use the absolute path `/app/bin/keepassxc-proxy` at the end, as `flatpak-spawn` does not inherit `$PATH` correctly.

```bash
#!/bin/bash
APP_REF="org.keepassxc.KeePassXC/x86_64/stable"
for inst in "$HOME/.local/share/flatpak" "/var/lib/flatpak"; do
    if [ -d "$inst/app/$APP_REF" ]; then
        FLATPAK_INST="$inst"
        break
    fi
done
[ -z "$FLATPAK_INST" ] && exit 1

APP_PATH="$FLATPAK_INST/app/$APP_REF/active"
RUNTIME_REF=$(awk -F'=' '$1=="runtime" { print $2 }' < "$APP_PATH/metadata")
RUNTIME_PATH="$FLATPAK_INST/runtime/$RUNTIME_REF/active"

exec flatpak-spawn \
  --env=LD_LIBRARY_PATH=/app/lib \
  --app-path="$APP_PATH/files" \
  --usr-path="$RUNTIME_PATH/files" \
  -- /app/bin/keepassxc-proxy "$@"

```

Make the script executable:

```bash
chmod +x ~/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/keepassxc-proxy-wrapper.sh
```

## 3. Define the native messaging host

Brave relies on a JSON manifest to locate the KeePassXC proxy. The app normally creates this file for you, but it isn't aware of the correct locations for the Flatpak.

Create the manifest directory and file:

```bash
mkdir -p ~/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/NativeMessagingHosts/
nano ~/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/NativeMessagingHosts/org.keepassxc.keepassxc_browser.json
```

Paste the following JSON. You must replace `YOUR_USERNAME` with your actual user directory. **Important (Fedora Silverblue users):** Also replace `/home` with `/var/home`.

```json
{
    "allowed_origins": [
        "chrome-extension://pdffhmdngciaglkoonimfcmckehcpafo/",
        "chrome-extension://oboonakemofpalcgghocfoadofidjkkk/"
    ],
    "description": "KeePassXC integration with native messaging support",
    "name": "org.keepassxc.keepassxc_browser",
    "path": "/home/YOUR_USERNAME/.var/app/com.brave.Browser/config/BraveSoftware/Brave-Browser/keepassxc-proxy-wrapper.sh",
    "type": "stdio"
}
```


## 4. Finalize the connection

1. Open KeePassXC, unlock your database, and go to **Tools** -> **Settings** -> **Browser Integration**.
2. Check **Enable browser integration** and select **Brave**.
3. Open Brave, install the KeePassXC-Browser extension, click the icon, and select **Connect**. Provide a name to authorize the link.
