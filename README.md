# cloud-workstation

An on-demand **Ubuntu 24.04 workspace** that runs on a GitHub Actions runner and is reachable **only over [Tailscale](https://tailscale.com)** — your own private network. Nothing is exposed to the public internet, and no password is ever written to the run log.

Three ways in, launched from the **Actions** tab:

- **Desktop** — a Yaru-themed XFCE desktop **streamed with Sunshine** (software H.264) and viewed in **[Moonlight](https://moonlight-stream.org)**. A low-latency video stream, not RDP — this is the smooth one.
- **Shell** — a plain terminal over Tailscale SSH. Keyless, ready in about a minute.
- **Windows** — a real Windows VM over RDP. Installs itself first (~15–30 min), then answers on a fixed address.
- **Kali** — a full Kali Linux desktop over RDP (`mstsc`). The security toolkit with a GUI, on your tailnet.

Each lab has its own workflow and its own `concurrency` group, so you can run a Desktop, a Shell and a Windows box **at the same time** — they don't cancel each other. A second run of the *same* lab does cancel the older one.

Ephemeral by design: an Actions job is capped at 6 hours, then everything is destroyed. A throwaway workspace, not persistent hosting.

---

## One-time setup

1. **Sign up** at [tailscale.com](https://tailscale.com) — free, no card. Install the client on your PC and sign in; leave it running.
2. **Generate an auth key**: [admin console → keys](https://login.tailscale.com/admin/settings/keys) → **Generate auth key** → tick **Ephemeral** + **Reusable**, **leave it untagged** → copy.
3. **Add it as a repo secret**: **Settings → Secrets and variables → Actions → New repository secret** → name `TS_AUTHKEY`, value the key.
4. *(Desktop only, optional)* add a `LAB_PASSWORD` secret to pick your own RDP password. Skip it and one is generated per run, readable only over Tailscale (see below).
5. **Allow SSH in your ACL** — [admin → Access controls](https://login.tailscale.com/admin/acls). New tailnets ship with this already; add it if SSH is refused:

   ```json
   "ssh": [
     { "action": "accept",
       "src":    ["autogroup:member"],
       "dst":    ["autogroup:self"],
       "users":  ["autogroup:nonroot", "root"] }
   ]
   ```

   `autogroup:self` only matches machines **you** own — which is why the auth key in step 2 must be **untagged**.

Full walkthrough and troubleshooting: **[setup.md](setup.md)**.

---

## Using it

**Desktop:** Actions → **"Desktop (Tailscale)"** → Run workflow. Ready in ~3–4 min (it installs XFCE + Sunshine). Then, with **[Moonlight](https://moonlight-stream.org)** installed on your PC:

1. In Moonlight, **Add PC manually** → `desktop.<tailnet>.ts.net`. It shows a 4-digit **PIN**.
2. Open `https://desktop.<tailnet>.ts.net:47990`, log in as `runner` (your `LAB_PASSWORD`, or the generated one via `ssh runner@desktop.<tailnet>.ts.net cat .lab-credentials`), and enter the PIN.
3. Click the PC → **Desktop**. In Moonlight settings, 1080p/60/~15–20 Mbps feels best.

The full pairing walkthrough is in **[setup.md](setup.md)**.

**Shell:** Actions → **"Shell (Tailscale)"** → Run workflow. Ready in ~1 min.
```bash
ssh runner@shell.<tailnet>.ts.net
```
Pick a **toolset** at launch (`base`, `security`, `dev`, `everything`). On `security`, the command `kali` opens a Kali rolling container sharing `~/lab` and the host network.

**Windows:** Actions → **"Windows (Tailscale)"** → Run workflow (pick an edition). It installs itself first, so the desktop answers after ~15–30 min — the Summary shows a ✅ ready line when it does. Then `mstsc` to `windows.<tailnet>.ts.net:3389`, sign in as `Docker` (same password rule as the desktop).

**Kali:** Actions → **"Kali (Tailscale)"** → Run workflow. First run pulls a multi-GB image (~5–10 min). Then `mstsc` to `kali.<tailnet>.ts.net:3389`, sign in as **`abc`** (the real Linux user, not `kali`), password per the usual rule. RDP only — no Moonlight.

## Stopping early

`touch ~/STOP` on the desktop (or `touch ~/lab/STOP` on the shell), or cancel the run from the Actions tab. Otherwise it auto-stops at the hours you chose. A new run of the same lab cancels an older one via its `concurrency` group.

---

## Why it's private

- **Tailscale only.** The RDP port and the SSH endpoint live on your tailnet; nothing is published to the internet, so the world-readable Actions log never carries a reachable address+credential pair.
- **No password in the log.** The RDP password is either your own secret or generated and delivered over the tailnet — it is masked and never printed.
- **Your devices only.** The tailnet ACL decides who connects, not this repo. Forked pull requests never receive `TS_AUTHKEY`, and `workflow_dispatch` needs write access, so only you can start a box.

## Notes

- **Nothing persists** between runs.
- **Same kernel as the host.** These are containers/processes on an Azure-hosted Ubuntu runner (`uname -r` shows an `-azure` kernel), not a separate VM. The Ubuntu userland and tools are the real thing.
- **Use it for what it's for.** GitHub Actions is meant for building, testing and deploying the repo's own software; a long-lived remote workspace is a grey area under GitHub's Acceptable Use Policies. Keep sessions short and occasional, and keep the work legitimate and authorized.

```
.github/workflows/desktop.yml   # XFCE desktop over Tailscale RDP
.github/workflows/shell.yml     # Ubuntu shell over Tailscale SSH
.github/workflows/windows.yml   # Windows VM (dockur/windows) over Tailscale RDP
.github/workflows/kali.yml      # Kali Linux desktop (linuxserver image) over Tailscale RDP
windows/install.bat             # unattended OEM tuning for the Windows VM
setup.md
README.md
```
