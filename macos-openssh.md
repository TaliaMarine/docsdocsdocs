# Enabling SSH (OpenSSH) on macOS for remote login

Goal: log in to a Mac ("the server") from another machine ("the workstation") over SSH
as the server's local user, using key-based authentication.

OpenSSH is already installed on macOS. You don't install it, you only turn on the
**Remote Login** service and then harden it.

## 0. Placeholders

| Placeholder | Meaning | Example |
|---|---|---|
| `<USER>` | Local account name on the server | output of `whoami` on the server |
| `<SERVER_HOST>` | Hostname or IP of the server | `modelmac.local` or `192.168.1.50` |

---

## 1. Server: enable Remote Login

### Option A: GUI

System Settings → General → Sharing → turn on **Remote Login**. Under the (i) button,
set "Allow access for" to **Only these users** and add your account.

### Option B: command line

```bash
sudo systemsetup -setremotelogin on
sudo systemsetup -getremotelogin
```

If `systemsetup` complains about needing Full Disk Access, grant it to your terminal app
under System Settings → Privacy & Security → Full Disk Access, or use the GUI option.

Alternative that needs no special permission:

```bash
sudo launchctl enable system/com.openssh.sshd
sudo launchctl bootstrap system /System/Library/LaunchDaemons/ssh.plist
```

Check that it listens on port 22:

```bash
sudo lsof -iTCP:22 -sTCP:LISTEN
```

## 2. Server: find its address

```bash
whoami                               # <USER>
scutil --get LocalHostName           # e.g. modelmac → reachable as modelmac.local
ipconfig getifaddr en0               # Wi-Fi IP (try en1 or others for Ethernet)
```

Reserve a fixed IP for this Mac in your router (DHCP reservation) so the address does
not change.

If the macOS firewall is on (System Settings → Network → Firewall), the built-in
`sshd` is allowed automatically when Remote Login is enabled. If connections still
fail, check Firewall options.

## 3. Workstation: first connection (password)

```bash
ssh <USER>@<SERVER_HOST>
```

Accept the host fingerprint on first connect (`yes`). Log in with the user's password to
confirm it works, then continue with keys.

## 4. Workstation: create a key and install it

```bash
# create a key (skip if you already have ~/.ssh/id_ed25519)
ssh-keygen -t ed25519 -C "workstation-to-server"

# copy the public key to the server
ssh-copy-id <USER>@<SERVER_HOST>
```

`ssh-copy-id` ships with recent macOS. If it's missing:

```bash
cat ~/.ssh/id_ed25519.pub | ssh <USER>@<SERVER_HOST> \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Test (it should not ask for the account password; it may ask for the key passphrase):

```bash
ssh <USER>@<SERVER_HOST>
```

Add the key to the macOS keychain agent so you type the passphrase once:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

## 5. Workstation: shortcut in `~/.ssh/config`

```
Host modelmac
    HostName <SERVER_HOST>
    User <USER>
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    UseKeychain yes
    ServerAliveInterval 30
```

Then simply:

```bash
ssh modelmac
```

---

## 6. Server: harden (do this after key login works)

Keep your current SSH session open while changing config, so a mistake doesn't lock you
out. macOS reads drop-in files from `/etc/ssh/sshd_config.d/`, which survive OS updates
better than editing `/etc/ssh/sshd_config`.

```bash
sudo tee /etc/ssh/sshd_config.d/100-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers <USER>
EOF

# validate syntax; no output means OK
sudo sshd -t

# apply
sudo launchctl kickstart -k system/com.openssh.sshd
```

Open a **second** terminal and confirm you can still log in before closing the first.

Notes:

- Password logins are now refused. If you lose your key, you need physical (or screen
  sharing) access to fix `~/.ssh/authorized_keys`.
- Check the effective settings with `sudo sshd -T | grep -Ei 'passwordauth|permitroot|allowusers'`.
- Some macOS versions ignore drop-ins if `/etc/ssh/sshd_config` lacks
  `Include /etc/ssh/sshd_config.d/*`. Check with `grep Include /etc/ssh/sshd_config`.

## 7. Keep the server reachable

A sleeping Mac drops SSH connections. For a plugged-in machine that should stay on:

```bash
sudo pmset -a sleep 0 disksleep 0 displaysleep 10 powernap 0
sudo pmset -a disablesleep 1      # stay awake with the lid closed
```

Undo with `sudo pmset -a disablesleep 0`. Wake-on-LAN ("Wake for network access" in
System Settings → Battery/Energy) also helps with Ethernet connections.

## 8. Optional

- **Different port:** add `Port 2222` to the drop-in file. macOS launches sshd through
  launchd socket activation, so also edit `/System/Library/LaunchDaemons/ssh.plist`
  (needs SIP changes) — usually not worth it. Keep port 22 and rely on keys.
- **Fail2ban-style protection:** not needed with passwords disabled.
- **Access from outside your LAN:** do **not** port-forward 22 to the internet. Use
  Tailscale or WireGuard and ssh to the tailnet name instead.
- **Copy files:** `scp file modelmac:~/` or `rsync -avz dir/ modelmac:~/dir/`.
- **Port forwarding:** `ssh -L 8080:localhost:8080 modelmac` makes the server's
  `localhost:8080` available at the workstation's `localhost:8080`.

## 9. Troubleshooting

| Symptom | Check |
|---|---|
| `Connection refused` | Remote Login off, or wrong host/IP; `sudo lsof -iTCP:22 -sTCP:LISTEN` on server |
| `Operation timed out` | different network/VLAN, firewall, or the server is asleep |
| `Permission denied (publickey)` | key not in `~/.ssh/authorized_keys`; permissions must be `700` on `~/.ssh`, `600` on `authorized_keys`, and the home directory not group/world-writable |
| User not allowed | user not in the Remote Login allowed list (Sharing settings) or not in `AllowUsers` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | server was reinstalled; `ssh-keygen -R <SERVER_HOST>` then reconnect |
| `.local` name not resolving | use the IP address |
| Debug | on workstation `ssh -vvv modelmac`; on server `sudo log stream --predicate 'process == "sshd"' --info` |
