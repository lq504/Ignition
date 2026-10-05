# Ignition (🔥+🪵=🗽)

**One command · One Microsoft login · Zero repeated auth**  
Hours of uninterrupted access to **NYU Torch** from your terminal and IDE.

**Two goals:**

1. **Get through Microsoft auth once.** The HPC login node requires the Microsoft device-code flow (open a URL, enter a PIN). This script automates that, making the first connection effortless.
2. **Keep one SSH connection alive and reuse it.** We set up a *persisting* underlying SSH connection (via `ControlMaster` / `ControlPersist` in your SSH config). For a long time afterward, every new session—another terminal, VS Code Remote, etc.—reuses that same connection. No separate auth run, no extra PIN. That’s why the config below is involved.

## Changes in this fork

- `ignite` now waits for SSH to report the login result instead of guessing with a timer: success is detected when SSH backgrounds itself, and an unfinished browser login is detected when Torch prints a new PIN (which is copied to the clipboard for you). This fixes false "Auth may have failed" messages when Torch is slow to confirm.
- SSH config: dropped `ForwardAgent yes` (not needed, and it exposes your SSH agent on a shared login node) and the duplicate `ServerAliveInterval`; explained why host-key checking is off.
- Removed `ignite-sh` (the password + Duo variant for NYU Shanghai / other HPCs) and its guide `NYUSHHPC.md`; this fork is Torch-only.
- README defaults now match the script; added troubleshooting for issues hit in practice (host key changes, an already-running connection, macOS Accessibility, VS Code server installs on Torch).

## Platforms

- [x] macOS
- [ ] Linux
- [ ] Windows

Fully supported on macOS, but Linux and Windows are untested. Any help with testing on Linux and Windows would be appreciated.

## Prerequisites

- **Expect** — `brew install expect` (macOS) or your distro’s `expect` package (Linux). On Windows you’d need Expect for Windows or WSL.
- **SSH** — Standard OpenSSH client.
- **Platform (for paste + Enter):**
  - **macOS:** built-in (no extra install).
  - **Linux:** `xdotool` (X11) or `ydotool` (Wayland) for sending keys to the browser.
  - **Windows:** PowerShell (built-in).

## 1. SSH config

The config below does two things: it defines the *torch* hpc host, and it turns on **connection sharing**. That way, after you run `ignite` once, every other SSH connection to the same host (terminal, VS Code, Cursor, etc.) reuses the same authenticated session—no extra auth. The extra options are what make that persistence work.

Create the control socket directory, then add the block:

```bash
mkdir -p ~/.ssh/cm
```

Then add (adjust `User` to your NYU NetID):

```
Host torch
  HostName login.torch.hpc.nyu.edu
  User YOUR_NETID
  # Torch has several login nodes with different host keys (see note below)
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  LogLevel ERROR
  # Reuse one authenticated connection for all sessions
  ControlMaster auto
  ControlPersist 8h
  ControlPath ~/.ssh/cm/%C
  ServerAliveInterval 60
  ServerAliveCountMax 3
```

> **Why host-key checking is off:** `login.torch.hpc.nyu.edu` resolves to one of several login nodes, each with its own host key, so strict checking fails with *"REMOTE HOST IDENTIFICATION HAS CHANGED"* whenever you land on a different node. NYU's own [connection guide](https://services.rt.nyu.edu/docs/hpc/connecting_to_hpc/connecting_to_hpc/) recommends these settings. The trade-off is that SSH no longer detects an impostor server; on the NYU network/VPN, with Microsoft auth on top, that risk is small. `ForwardAgent` is deliberately left out — Torch doesn't need it.

- Replace `YOUR_NETID` with your Torch username (e.g. `yl22`).

**What those options do (briefly):**

| Option | Purpose |
|--------|--------|
| `StrictHostKeyChecking no` + `UserKnownHostsFile /dev/null` | Don't fail when you land on a different login node with a different host key. |
| `LogLevel ERROR` | Hide the "Permanently added … to known hosts" notice that would otherwise appear on every connection. |
| `ControlMaster auto` | Reuse an existing SSH connection when you run `ssh torch` again instead of opening a new one. |
| `ControlPersist 8h` | Keep the shared connection alive for 8 hours after the last session, so later logins skip device auth. |
| `ControlPath ~/.ssh/cm/%C` | Path for the connection socket; `%C` is a hash so each host gets its own socket under `~/.ssh/cm/`. |
| `ServerAliveInterval 60` | Send a keep-alive every 60 seconds so the connection isn’t dropped by firewalls when idle. |
| `ServerAliveCountMax 3` | After 3 unanswered keep-alives, treat the connection as dead and disconnect. |

## 2. Run

### 2.1 Make the script executable:
Once, from this repo’s directory
```bash
cd path/to/ignite
chmod +x ignite
```

### 2.1.5 Optional — run `ignite` from anywhere:

**Option A — Add this directory to PATH** (if the repo is e.g. `~/bin`):

```bash
# In ~/.zshrc or ~/.bashrc
export PATH="$PATH:$HOME/bin/Ignition"
```

Then reload your shell (`source ~/.zshrc` or open a new terminal). You can run `ignite` from any directory.

**Option B — Symlink into a directory already on PATH** (e.g. `/usr/local/bin`):

```bash
ln -sf "$(pwd)/ignite" /usr/local/bin/ignite
```

(Use the full path to this repo if you’re not in it.) Then `ignite` is available everywhere.

### 2.2 Run `ignite`:

```bash
ignite
```

(From this directory, or from any directory if you added it to PATH.)

**What happens:**

1. SSH starts to **torch**.
2. On “Authenticate with PIN …”, the script opens the login URL and copies the PIN to the clipboard.
3. ~0.5s later it sends **Paste** + **5× Enter** to the focused window — keep the browser in front.
4. After `browser_auth_wait` seconds (default 5), it sends **Enter** to SSH and waits (up to 120s) for the result:
   - **Success:** SSH moves to the background; the script confirms the shared connection with `ssh -O check torch` and prints *“SSH session ready. You're in.”*
   - **Browser login not finished:** Torch prints a new PIN. The script copies it to the clipboard and hands you the session — enter the PIN in the browser, finish signing in, then press Enter in the terminal.

> **macOS permission:** the simulated Paste/Enter needs Terminal (or iTerm) to be allowed under *System Settings → Privacy & Security → Accessibility* (and *Automation → System Events* if listed). Without it, the keystrokes silently do nothing — paste the PIN yourself.

### 2.3 Verbosity:
- `quiet` or `q` (only errors and outcome)
- `verbose` or `v` (default)
- `vv` (very verbose)

**Examples:**
```bash
ignite quiet
ignite vv
```

(Options use words like `quiet` instead of `-q` because Expect treats leading dashes as its own flags.)


## 3. Using the connection from VS Code / Cursor

Once you’ve run `ignite` and the connection is up, that same connection is reused by anything that uses your SSH config.

- **VS Code / Cursor Remote-SSH:** Use “Connect to Host…” (or the Remote Explorer) and choose **torch** (the same host name from your config). They will use the existing connection; you won’t be prompted for device auth again until the connection expires (e.g. after 8 hours or a reboot).
- **Another terminal:** Run `ssh torch` (or `ssh torch` and then your usual commands). Same underlying connection.
- **Other tools:** Anything that uses `ssh torch` (e.g. `rsync`, `scp`, Git over SSH) will also reuse the connection as long as it’s still alive.

So: run `ignite` once to get through Microsoft auth and establish the persistent connection; then use **torch** as the host everywhere. The “cumbersome” config is what makes this possible.

## 4. Tuning

1. **Delay before sending Enter to SSH:** Change `set browser_auth_wait 5` (near the top) to the number of seconds you generally need to finish login in the browser (e.g., `8` if MFA or the account picker slows you down). If you're too slow, the script now detects it and gives you a new PIN rather than failing.
2. **Delay between opening the URL and simulating Paste:** Change `set paste_delay 0.5` (default is 0.5 seconds; try `1.5` if the paste lands before the page has loaded) if you want to make the script paste the PIN sooner or give yourself more time to switch focus to your browser.
3. **Interval between each simulated Enter after Paste:** Change `set press_enter_interval 1` (default is 1 second) to adjust how quickly each Enter key press is sent after pasting.

All these settings appear near the top of `ignite`. Adjust them if you want to make the flow faster or slower for your own pace.

## Notes

- The script uses your **default browser** and simulates Paste + Enter; it does not install or drive a separate browser.
- `ignite` is only needed when no shared connection is running. Check with `ssh -O check torch` (*“Master running”* means you're already connected); close it early with `ssh -O exit torch`. A one-liner that only logs in when needed:
  ```bash
  alias torch-up='ssh -O check torch 2>/dev/null || ignite'
  ```
- Without `ignite`, plain `ssh torch` (or VS Code connecting to `torch`) also works: enter the PIN by hand and that session becomes the shared connection.
- **VPN:** `login.torch.hpc.nyu.edu` only resolves on the NYU network/VPN. If the VPN drops, the shared connection dies with it — reconnect and run `ignite` again.

## FAQ
**Q:** Why does my terminal show “SSH session ready” while the browser reports an invalid PIN?

**A:** This usually happens due to **timing**. The browser authentication page may not have fully refreshed before the SSH login completed. This is often caused by **network latency** or **insufficient wait time** between automated steps. Increasing the delay between operations and restarting the authentication flow typically resolves the issue.

**Q:** `ignite` says *“SSH exited before PIN prompt”* even though my Internet/VPN is fine.

**A:** You most likely already have a shared connection, so SSH has nothing to log in to. `ssh -O check torch` will say *“Master running”*. Just use `ssh torch` (or the `torch-up` alias above).

**Q:** `ignite` hangs silently on the very first run.

**A:** SSH is waiting on a hidden host-key question (*“Are you sure you want to continue connecting?”*). This only happens if host-key checking is still on — use the config above, or run `ssh torch` once by hand and answer `yes`.

**Q:** *“WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!”*

**A:** You landed on a different login node. Add `StrictHostKeyChecking no` and `UserKnownHostsFile /dev/null` to the `torch` entry (see the config above), then remove the stale keys with `ssh-keygen -R login.torch.hpc.nyu.edu`.

**Q:** VS Code Remote-SSH hangs at *“Downloading/Extracting VS Code Server”* or fails with *`ServerNotExecutable … code-server is not executable`*.

**A:** Torch's login node may not reach Microsoft's download server, and unpacking onto the network home directory is slow, so automatic installs can end half-finished. Install the server by hand: get the commit from *Code → About*, then

```bash
C=<commit>
curl -fL -o /tmp/vscode-server.tgz "https://update.code.visualstudio.com/commit:$C/server-linux-x64/stable"
scp /tmp/vscode-server.tgz torch:
ssh torch "D=\$HOME/.vscode-server/cli/servers/Stable-$C; mkdir -p \$D/server && tar -xzf ~/vscode-server.tgz --strip-components 1 -C \$D/server && ls -l \$D/server/bin/code-server"
```

Unpacking can take several minutes. Also disable the *Dev Containers* extension if it's installed — it can start a second, competing install. Setting `"update.mode": "manual"` in VS Code avoids surprise server re-installs after updates. If extension installs stall, add `"remote.downloadExtensionsLocally": true`.

## Acknowledgements

- The use of `expect` was inspired by [JosiahWayne](https://github.com/JosiahWayne/torch-login). I spent a long time trying to extract the device code from an interactive terminal session, and this approach solves that problem cleanly and reliably.
- Many thanks to the NYU HPC team for their continued efforts in providing a robust research computing platform for the NYU community. While Torch can be challenging to use at times, I have confidence that the system will continue to improve thanks to the dedication and expertise of the team.
