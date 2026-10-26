# Connecting to CYB601 Bastion via SSH

Your username is **pgsXX** and your starting password is **`ChangeMe2026!`**

---

## Step 1: Install cloudflared

**Windows (PowerShell):**

```powershell
winget install --id Cloudflare.cloudflared
```

Close and reopen PowerShell afterwards.

**macOS:**

```bash
brew install cloudflared
```

**Linux (Debian/Ubuntu):**

```bash
curl -L -o cloudflared.deb <https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb>
sudo dpkg -i cloudflared.deb
```

---

## Step 2: Create your SSH config

**Windows (PowerShell):**

```powershell
mkdir $HOME\.ssh -Force
notepad $HOME\.ssh\config
```

Paste this, then save. Make sure the file isn't saved as `config.txt`.

```
Host cyb601-bastion
    HostName cyb601.aatizghimire.com.np
    User pgsXX
    ProxyCommand "C:\Program Files (x86)\cloudflared\cloudflared.exe" access ssh --hostname %h
```

**macOS / Linux:**

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/config
```

Paste this, then save with `Ctrl+O`, `Enter`, `Ctrl+X`:

```
Host cyb601-bastion
    HostName cyb601.aatizghimire.com.np
    User pgsXX
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
```

> Run `which cloudflared` and use the path it shows. It's often `/usr/bin/cloudflared` on Linux and `/opt/homebrew/bin/cloudflared` on Apple Silicon Macs.
> 

---

## Step 3: Log in and change your password

```
ssh cyb601-bastion
```

Type `yes` if it asks about the fingerprint, then enter `ChangeMe2026!`. Nothing appears on screen while you type, which is normal.

Once you're in, change your password right away:

```bash
passwd
```

Then type `exit` to log out.

---

## Step 4: Generate your SSH key (on your own computer)

The command is the same on all systems:

```
ssh-keygen -t ed25519
```

Press `Enter` to accept the default location. You can set a passphrase for extra safety or press `Enter` twice to skip it.

This creates two files in your `.ssh` folder:

- `id_ed25519` is your **private key**. Never share it with anyone.
- `id_ed25519.pub` is your **public key**, and that's the one that goes on the server.

---

## Step 5: Upload your public key

**macOS / Linux:**

```bash
ssh-copy-id cyb601-bastion
```

**Windows (PowerShell):**

```powershell
type $HOME\.ssh\id_ed25519.pub | ssh cyb601-bastion "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Enter your new password when asked.

---

## Step 6: Test it

```
ssh cyb601-bastion
```

You should log in **without** a password prompt (or only your key passphrase, if you set one). You're done! 🎉

---

## Troubleshooting

**"no such host"**: Flush your DNS cache and try again.

| OS | Command |
| --- | --- |
| Windows | `ipconfig /flushdns` |
| macOS | `sudo dscacheutil -flushcache` |
| Linux | `resolvectl flush-caches` |

**"Permission denied" with your password**: Check that Caps Lock is off and type the password carefully, including the `!`.

**Still asked for a password after Step 5**: The key didn't upload. Repeat Step 5.