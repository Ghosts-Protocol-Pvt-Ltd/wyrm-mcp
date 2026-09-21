# Troubleshooting `npm install -g wyrm-mcp`

The native `better-sqlite3` module is the only piece that can fail at install time. This guide walks the three failure modes we've seen in the wild, in order from most-common to least.

If your installation succeeded **and `wyrm` actually runs**, you don't need this page.

---

## 0. `Could not locate the bindings file` — npm v12+ silently skipped the native build

**Symptom:** `npm install -g wyrm-mcp` reports success, but running `wyrm` (or any DB operation) crashes with `Error: Could not locate the bindings file ... better_sqlite3.node`.

**Cause:** npm v12 changed the default to **deny dependency lifecycle scripts** (an allow-list security feature — `ignore-scripts` is still `false`). So `better-sqlite3`'s native addon never compiles, and `wyrm-mcp`'s own postinstall is skipped too. The install "succeeds" with an easy-to-miss `npm warn install-scripts` line.

**Fix** — reinstall with the two packages' build scripts allow-listed:

```bash
npm install -g wyrm-mcp --allow-scripts=wyrm-mcp,better-sqlite3,onnxruntime-node,protobufjs
```

Or make it the default for all your global installs:

```bash
npm config set allow-scripts=wyrm-mcp,better-sqlite3,onnxruntime-node,protobufjs --location=user
```

`wyrm update` passes `--allow-scripts` automatically, so once you're installed correctly, updates won't hit this.

The list names every package in the install that has an install script: Wyrm's own, `onnxruntime-node` and `protobufjs`. `better-sqlite3` stays on it for older installs; from version 13 its native binding ships prebuilt inside the package and it has no install script.

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
| **Windows** | Install Visual Studio Build Tools with the "Desktop development with C++" workload, plus Python 3. The Node.js installer's "Tools for Native Modules" option sets up both. (The old `windows-build-tools` npm package is deprecated and does not work with current Node.) |
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

## 6. Windows: `wyrm-setup` fails with `__dirname is not defined`, or every session says `wyrm CLI not found`

Both were bugs in 9.2.3 and earlier, reported from Windows 11 (Node 24, npm 11, Claude Code) and fixed in the release after 9.2.3. The MCP server, the `wyrm` CLI and `wyrm doctor` were not affected, which is why it looked healthy from a terminal.

**`Setup failed: ReferenceError: __dirname is not defined`.** Setup looked for the server with `which` and a bare `npm`, neither of which can run on Windows, then hit a line that does not work in an ES module. On 9.2.3, pass the server path yourself:

```powershell
wyrm-setup only claude-code --plain --server "$env:APPDATA\npm\node_modules\wyrm-mcp\dist\index.js"
```

**`Wyrm unavailable: wyrm CLI not found. Nothing was restored.`** at the start of every Claude Code session, while `wyrm` works in any terminal. The session hooks launched a bare `wyrm`. On Windows that is a `.cmd` shim, which Node will not start without a shell, so nothing was captured or restored. Upgrade, then run `wyrm setup` once more. The hooks now start the CLI's own `dist/wyrm.js` with their own `node`, with no shim and no `PATH` lookup: first from the install that belongs to the running Node, then from npm's default prefix, and last from the path setup records in `~/.claude/hooks/wyrm-cli.json` (which is what finds a custom prefix, Volta, Scoop or pnpm). To check it without opening Claude Code:

```powershell
'{}' | node "$env:USERPROFILE\.claude\hooks\wyrm-session-rehydrate.mjs"
```

The answer is one line of JSON. A healthy `systemMessage` contains `Wyrm:` or `Wyrm restored`. A broken one contains `Wyrm unavailable`.

**Red `NativeCommandError` blocks during setup in Windows PowerShell 5.1.** PowerShell 5.1 renders anything a program writes to stderr as an error. Wyrm's informational log lines go to stderr, so a successful setup looked like a failure. Setup now logs warnings and errors only, like the rest of the CLI. `$env:WYRM_LOG_LEVEL='info'` brings the lines back.

**npm skipped the install scripts.** See section 0. This was seen on npm 11.17 as well as npm 12. In PowerShell, quote the flag so the commas survive on older npm shims: `npm install -g wyrm-mcp "--allow-scripts=wyrm-mcp,better-sqlite3,onnxruntime-node,protobufjs"`.

**`wyrm-setup` cannot be loaded because running scripts is disabled on this system.** That is PowerShell's execution policy refusing npm's `.ps1` shim, not Wyrm. Run the `.cmd` shim instead: `wyrm-setup.cmd`, `wyrm.cmd`, `npm.cmd`.

**`wyrm update` fails with `EPERM` or `EBUSY`.** Windows will not replace a file that is open, and a running Wyrm MCP server keeps the native SQLite module open. Close Claude Code and any editor running the server, then update. For the same reason the `wyrm_self_update` tool does not run on Windows; it prints the command instead.

**The statusline.** On Windows it is computed in-process on every render. There is no statusline daemon there: the only endpoint it could listen on is a named pipe, which another local user could claim first.

---

## Still stuck?

Open an issue at https://github.com/Ghosts-Protocol-Pvt-Ltd/wyrm-mcp/issues with:
- The full `npm install` output (especially the `gyp ERR!` lines)
- `node --version`, `npm --version`, `uname -srmo`
- The output of `cat /etc/os-release` (Linux) or `sw_vers` (macOS)
