# Troubleshooting `npm install -g wyrm-mcp`

The native `better-sqlite3` module is the only piece that can fail at install time. This guide walks the three failure modes we've seen in the wild, in order from most-common to least.

If your installation succeeded, you don't need this page.

---

## 1. `cc: error: unsupported argument '4' to option '-flto='` — Termux / proot-distro on Android

You're on Termux + a `proot-distro` Linux guest (Fedora, Ubuntu, Debian). Node's bundled `common.gypi` ships `-flto=4`, and the Bionic-derived toolchain doesn't accept the numeric form.

**Fix (automatic since 5.2.1):** the `preinstall` hook patches the gypi for you. If the patch didn't run for some reason, run it manually:

```bash
node /usr/local/lib/node_modules/wyrm-mcp/scripts/preinstall.cjs
npm install -g wyrm-mcp     # try again
```

To opt out of the patch (won't fix the failure, but won't touch your gypi either): `WYRM_SKIP_TERMUX_FIX=1 npm install -g wyrm-mcp`.

---

## 2. `prebuild-install warn install Request timed out`

`prebuild-install` fetches a prebuilt `better-sqlite3` binary from GitHub releases; if the request times out (slow connection, GitHub flakiness, corporate proxy), it falls back to a source build. If you have a C/C++ toolchain installed, the source build succeeds. If you don't, see section 3.

**Quick fixes:**

```bash
# Longer fetch timeout (5 min):
npm config set fetch-timeout 300000
npm config set fetch-retry-maxtimeout 600000
npm install -g wyrm-mcp

# Or use a different DNS so GitHub releases resolve faster:
echo 'nameserver 1.1.1.1' | sudo tee /etc/resolv.conf
```

If the network is genuinely cut off (air-gapped install): download `wyrm-mcp-X.Y.Z.tgz` from `npm pack wyrm-mcp` on a machine with internet, copy the tarball over, then `npm install -g ./wyrm-mcp-X.Y.Z.tgz`.

---

## 3. `gyp ERR! find Python` / `make: command not found` / `cc: command not found`

Source build was attempted but the host has no compiler. Either you're on an exotic architecture with no prebuilt available (s390x, ppc64le, riscv64), or `prebuild-install` failed (section 2) and the fallback compile can't run.

**Fix:** install a C/C++ toolchain + `make` + `python3`. The preinstall hook detects this and prints the right command for your distro, but if you missed it:

| Distro / OS | Command |
|------|------|
| **Debian / Ubuntu / Mint / Kali / Pop!_OS / Raspbian** | `sudo apt-get update && sudo apt-get install -y build-essential python3` |
| **Fedora / RHEL / CentOS / Rocky / Alma** | `sudo dnf install -y gcc gcc-c++ make python3` |
| **Alpine** | `sudo apk add --no-cache build-base python3` |
| **Arch / Manjaro** | `sudo pacman -S --needed base-devel python` |
| **openSUSE** | `sudo zypper install -y gcc gcc-c++ make python3` |
| **macOS** | `xcode-select --install` (provides clang + make); `brew install python3` |
| **Windows** | `npm install --global windows-build-tools` (or install Visual Studio Build Tools manually) |
| **Termux / proot-distro Fedora** | `dnf install -y gcc gcc-c++ make python3 sqlite-devel` (inside the proot guest) |
| **Docker `node:alpine`** | base image lacks build tools: `RUN apk add --no-cache build-base python3` in your Dockerfile before `npm install` |

After install, retry: `npm install -g wyrm-mcp`.

---

## 4. Bun / Deno / WebContainer / Stackblitz

Wyrm uses Node's native module API (`process.dlopen` → C++ N-API). Bun and Deno emulate some of this but not all of it; WebContainer can't run native modules at all. There's no workaround for these environments at install time.

If you need Wyrm in one of these, run it as a separate process and talk to it over HTTP:

```bash
wyrm serve &                     # start the HTTP API on :3333
# then from Bun/Deno, fetch http://localhost:3333 endpoints
```

---

## 5. Permission denied during global install

```
npm ERR! EACCES: permission denied, ... '/usr/local/lib/node_modules'
```

Don't `sudo npm install` — fix your npm prefix instead so you never need root:

```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g wyrm-mcp
```

---

## Still stuck?

Open an issue at https://github.com/Ghosts-Protocol-Pvt-Ltd/Wyrm/issues with:
- The full `npm install` output (especially the `gyp ERR!` lines)
- `node --version`, `npm --version`, `uname -srmo`
- The output of `cat /etc/os-release` (Linux) or `sw_vers` (macOS)
