# Setup Guide

Two workspaces, both private over Tailscale. This page is the full walkthrough and the fixes for the things that usually trip people up.

---

## First: Tailscale (needed by both)

1. **Sign up** at [tailscale.com](https://tailscale.com) — free, no card.
2. **Install Tailscale** on your PC (and phone, if you want in from there). Sign in, leave it running — the tray icon should say *Connected*.
3. **Generate an auth key**: [login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys) → **Generate auth key**:
   - tick **Ephemeral** (so a finished run's node disappears instead of piling up),
   - tick **Reusable** (so every run can use the same key),
   - **leave it untagged** — this matters, see the ACL note below,
   - copy the key.
4. **Add the repo secret**: repo → **Settings → Secrets and variables → Actions → New repository secret** → name `TS_AUTHKEY`, paste the key.
5. **Allow SSH in your ACL**: [login.tailscale.com/admin/acls](https://login.tailscale.com/admin/acls). Most new tailnets already include this in the sample policy. If SSH is ever refused, this is why:

   ```json
   "ssh": [
     {
       "action": "accept",
       "src":    ["autogroup:member"],
       "dst":    ["autogroup:self"],
       "users":  ["autogroup:nonroot", "root"]
     }
   ]
   ```

   > `autogroup:self` only matches machines **you** own. A **tagged** auth key makes the *tailnet* the owner instead, and this rule stops matching — which is the whole reason step 3 says leave the key untagged.

---

## Desktop (Sunshine + Moonlight)

**Workflow**: Actions → **"Desktop (Tailscale)"** → Run workflow.

Ready in **~3–4 min** — it installs a Yaru-themed XFCE desktop plus **Sunshine**, which streams the screen as low-latency **H.264 video**. You watch it in **[Moonlight](https://moonlight-stream.org)** (free, install on your PC first). This is a video stream, not RDP — that's why it feels smooth even far from the runner. The runner has no GPU, so encoding is done in software; XFCE is kept light so the CPU has room to encode.

### The password (this is the private bit)

The Sunshine web UI (used once, to pair) logs in as `runner`. The password never reaches the public run log:

| | Password | Setup |
|---|---|---|
| **`LAB_PASSWORD` secret** *(recommended)* | the one you chose | add a `LAB_PASSWORD` repo secret once |
| **Auto-generated** *(default)* | read over the tailnet: `ssh runner@desktop.<tailnet>.ts.net cat .lab-credentials` | nothing — it just works |

### Pair (once)

1. Install **Moonlight** on your PC and make sure **Tailscale is connected**.
2. In Moonlight → **Add PC manually** → enter `desktop.<tailnet>.ts.net`. It shows a 4-digit **PIN**.
3. Open **`https://desktop.<tailnet>.ts.net:47990`**, accept the self-signed cert, log in as `runner` (password above).
4. Go to the **PIN** tab, paste the PIN, **Send**. Paired.

### Stream

Back in Moonlight, click the PC → **Desktop**. For the best feel, in Moonlight **Settings**:

- **Resolution:** 1080p (or 720p if you're far from the runner — fewer pixels to encode)
- **FPS:** 60
- **Bitrate:** ~15–20 Mbps
- **V-Sync:** off; **frame pacing:** on

> If it feels CPU-bound (tearing, dropped frames), drop to 720p first — software encoding is the bottleneck on a GPU-less box, and halving the pixels helps most.

You also get a **shell on the same machine** any time: `ssh runner@desktop.<tailnet>.ts.net`.

---

## Windows (Tailscale RDP)

**Workflow**: Actions → **"Windows (Tailscale)"** → Run workflow. Pick an edition (`11l` = Windows 11 LTSC is the light default), RAM and cores.

This one boots a **real Windows VM** (KVM-accelerated QEMU), so it's the slowest to come up — Windows installs itself unattended, typically **15–30 min**. The Summary shows the address right away, but RDP won't answer until a **✅ ready** line appears.

- **Password:** same rule as the desktop — your `LAB_PASSWORD` secret, or generated and read over the tailnet (`ssh runner@windows.<tailnet>.ts.net cat .lab-credentials`).
- **Connect:** `mstsc` → `windows.<tailnet>.ts.net:3389` → username `Docker`.
- **Fast path:** Tailscale carries UDP, so RDP's bitmap caching + compression work at full speed. The VM is pre-tuned (`windows/install.bat`): AVC444 codec, animations off, flat wallpaper, outline drag. No client-side UDP tweak needed.

> The Windows lab is the heaviest and the most conspicuous — a nested KVM VM held open for hours is exactly the shape platform abuse-detection looks for. Use it deliberately and keep sessions short.

---

## Shell (Tailscale SSH)

**Workflow**: Actions → **"Shell (Tailscale)"** → Run workflow.

Ready in **~1 min**. The runner *is* Ubuntu 24.04, so this just joins it to your tailnet with Tailscale SSH and hands it to you.

```bash
ssh runner@shell.<tailnet>.ts.net
```

No password, no key file, no open port — `tailscaled` terminates the SSH session itself and authorises you by tailnet identity.

### Toolsets (choose at launch)

| Toolset | What you get |
|---|---|
| `base` | tmux, neovim, ripgrep, fzf, jq, htop, 7z |
| `security` | base + nmap, masscan, tcpdump, tshark, hydra, john, hashcat, sqlmap, nikto, binwalk… **+ a Kali rolling container** |
| `dev` | base + build-essential, python3-venv, pipx, uv, sqlite3, psql/redis clients |
| `everything` | all of the above |

On `security`/`everything`, type **`kali`** for a shell inside a Kali rolling container (`kali-linux-headless`), sharing `~/lab` and the host network so scans and captures behave natively. Kali's apt repo is deliberately *not* bolted onto Ubuntu — mixing them is the standard route to a broken package set.

---

## Ending a session early

`touch ~/STOP` on the desktop, or `touch ~/lab/STOP` on the shell, or cancel the run in the Actions tab. Otherwise it stops at the hours you chose (desktop default 3, shell default 5.5; max 5.7 either way, because Actions hard-kills at 6h).

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `mstsc`: "remote computer name not valid" | Tailscale isn't connected on your PC, or you're using an old address. Reconnect Tailscale; copy the address fresh from Summary. |
| RDP connects then drops at the login screen | Rare xrdp session race — reconnect once. Persistent: check the run's "Tune xrdp" step for the diagnostics dump. |
| `ssh`: "access denied" | Your tailnet ACL is missing the SSH rule above, **or** the auth key was tagged. Fix the ACL / regenerate an untagged key. |
| Registered as `desktop-1` not `desktop` | A previous node still holds the name; ephemeral keys reap it within minutes. Use the tailnet IP (in Summary) meanwhile. |
| Run fails immediately | `TS_AUTHKEY` secret is missing or expired. Regenerate and re-add it. |
