# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build System Architecture

This repo has **one canonical build script**: `scripts/local-build.sh`. All build paths (WSL2, GitHub Actions, coverage tests) invoke it directly — there is no second source of truth.

```
scripts/local-build.sh     # Linux / WSL2 / GitHub Actions — the only build entry point
scripts/local-build.ps1    # Windows + WSL2 wrapper (rsyncs repo to WSL native path)
scripts/coverage-test.sh  # Config coverage → scripts/local-build.sh --config-only (fast checks)
.github/workflows/         # GitHub Actions → scripts/local-build.sh
```

**Build entry point**: `scripts/local-build.sh`. Every ENTRYPOINT calls this. When modifying build logic, change only this file — do not add logic to wrapper scripts.

### Build Phases

`scripts/local-build.sh` executes in order:

1. **check_env** — validates Linux OS, disk space, required commands, creates `install` wrapper for uutils, symlinks host cmake 3.x to staging_dir, symlinks system gzip to staging_dir
2. **prepare_source** — clones `immortalwrt-mt798x-24.10` (branch `mt798x-mt799x-6.6-mtwifi`) if missing or mismatched branch; resets to origin HEAD
3. **prepare_feeds** — copies `feeds.conf.default`, stashes local feed modifications, runs `./scripts/feeds update -a`, restores stashed changes, installs feeds
4. **apply_package_fixes** — applies WiFi patch, patches QMI drivers, patches MosDNS/v2dat Go 1.24 compat, patches v2ray-geodata downloads, clones external LuCI i18n packages, patches HomeProxy WAN detection, installs libcrypt-compat stub, conditionally clones AdGuardHome/OpenClash/Adbyby-plus/MosDNS/HomeProxy feeds
5. **install_selected_packages** — runs `./scripts/feeds install -f` for enabled features; verifies required package Makefiles exist
6. **configure_build** — fetches base defconfig, appends `h5000m.extra.config` and inline WiFi-driver/EasyMesh/Nikki/MosDNS/HomeProxy/VLMCSd/QModem blocks, runs `make defconfig`, retries UPnP nftables stack if luci-app-upnp was auto-disabled, verifies all requested packages are `=y` in `.config`
7. **prefetch_and_toolchain** — runs `make download`, then `make world` to build toolchain (GCC 13.3.0) if not cached; applies GCC 16 char8_t S2C overload patch to libcody before toolchain build
8. **compile_firmware** — cleans Go module cache, precompiles MosDNS v2dat, runs `make firmware -jN`; falls back to single-threaded on failure with diagnostic output
9. **collect_artifacts** — copies firmware images, manifest, build.config, enabled-packages.txt to `artifacts/`

## Common Commands

```bash
# Full build (WSL2 on Windows — recommended for local)
powershell.exe -ExecutionPolicy Bypass -File ./scripts/local-build.ps1

# Config-only (fast iteration, no firmware compile)
powershell.exe -ExecutionPolicy Bypass -File ./scripts/local-build.ps1 -ConfigOnly

# Config coverage tests (validates multiple feature combinations)
bash scripts/coverage-test.sh quick        # default + proxy-stack profiles
PROFILE_SET=full bash scripts/coverage-test.sh  # all 11 profiles

# WSL direct (Linux host or inside WSL shell)
bash scripts/local-build.sh --install-deps
bash scripts/local-build.sh --config-only
SKIP_TOOLCHAIN=true SKIP_DOWNLOAD=true bash scripts/local-build.sh  # fast iteration
```

### Feature Switches (environment variables)

All controlled via `ENABLE_*` env vars passed through to `scripts/local-build.sh`. Key ones:

| Variable | Default | Notes |
|---|---|---|
| `ENABLE_NIKKI` | `true` | mihomo-meta proxy (Nikki). Conflicts with `ENABLE_MOSDNS` and `ENABLE_HOMEPROXY` (DNS stack overlap) |
| `ENABLE_MOSDNS` | `true` | DNS proxy with v2dat. Has Go 1.24 compatibility patches |
| `ENABLE_HOMEPROXY` | `false` | sing-box based proxy. Uses fwmark routing |
| `ENABLE_UPNP` | `true` | miniupnpd-nftables + luci-app-upnp (fw4/nftables only, no iptables) |
| `ENABLE_EASYMESH` | `true` | mesh11sd + wpad-mesh-openssl (replaces base wpad) |
| `SKIP_FEEDS_UPDATE` | `false` | Reuse existing feed checkouts |
| `SKIP_TOOLCHAIN` | `false` | Skip toolchain build |
| `SKIP_DOWNLOAD` | `false` | Skip source download |

**Conflicting/mutex features:**
- `ENABLE_QMODEM_NEXT` and `ENABLE_QMODEM` are mutually exclusive
- `ENABLE_ORIGINAL_MODEM` conflicts with both QModem options
- `ENABLE_NIKKI`, `ENABLE_HOMEPROXY`, `ENABLE_MOSDNS` each bring their own DNS hijack/TPROXY setup

## Code Architecture

### Scripts
- **`scripts/local-build.sh`** — The only build entry point. See Build Phases above.
- **`scripts/coverage-test.sh`** — Runs named profiles (`default`, `minimal`, `proxy-stack`, `homeproxy-only`, `mosdns-only`, `nikki-only`, `qmodem-legacy`, `original-modem`, `optional-services`, `all-compatible`, `dockerman`) each calling `local-build.sh --config-only`. Profile set `quick` runs 2 profiles; `full` runs all 11. Profiles run in parallel.
- **`scripts/local-build.ps1`** — WSL2-based Windows build wrapper. Rsyncs repo to WSL native path (`~/Auto-H5000M-BIN-localbuild`) before building for case-sensitive filesystem compatibility. All `ENABLE_*` / mirror / repo env vars are forwarded automatically.

### Config Fragments
- **`feeds.conf.default`** — OpenWrt feeds: packages, luci, routing, telephony, telephony + optional qmodem and nikki third-party feeds.
- **`h5000m.extra.config`** — Kernel/config fragments appended after the base defconfig. Sets Argon theme, turboacc, Airpifanctrl, ccache, build user/domain, and disables wrtbwmon/rclone.

### Key Embedded Fixes
- **Go 1.24 compat** (`patch_mosdns_go124`, `patch_v2dat_go124`): MosDNS and v2dat upstream requires Go 1.25+; the script patches `go.mod`/Makefiles to downgrade to Go 1.24-compatible deps and strips fragile `go.mod` hunks from v2dat perf patches. Additionally replaces the standard `packages_lang_golang` feed with `sbwml/packages_lang_golang -b 24.x` to provide Go 1.24 toolchain.
- **HomeProxy** (`patch_homeproxy_no_wan_default_interface`): Fixes `auto_detect_interface` in `generate_client.uc` — when WAN is unavailable (`wan_status.up = false`), sets to `false` instead of `null` to prevent sing-box error `missing default interface`.
- **v2ray-geodata**: Patches `Build/Compile` in `package/v2ray-geodata/Makefile` to add ghfast/gh-proxy/gh.llkk.cc proxy URLs for geoip/geosite downloads.
- **QMI driver** (`fix_qmi_driver`): Patches QMI WWAN drivers (Fibocom/Quectel/Simcom) for Linux 6.6: replaces `u64_stats_fetch_begin_irq` → `u64_stats_fetch_begin` and `memcpy` → `eth_hw_addr_set`/`dev_addr_set`.
- **libcrypt-compat stub**: When the build references `libcrypt-compat` but feeds don't define it, creates a stub package so package dependency scanning stays consistent.
- **sanitize_path**: Creates GNU-invocation wrapper for uutils `install`, symlinks host cmake 3.x to staging_dir for Arch Linux hosts, symlinks system gzip to staging_dir if missing.

### Patches
- **`patches/mtwifi-apcli-active-only.patch`** — Persistent WiFi fix for MTK MT7992. Adds `cfg_is_true`, `vif_is_enabled`, `sorted_vif_indices` helpers; changes `dats.BssidNum` to use effective (non-disabled) VIF count; adds early-exit guards for `disabled="1"` VIFs in `netifd/mtwifi.sh`. Applied via forward/reverse dry-run guards. `verify_mtwifi_patch()` validates patch markers after application — if the upstream source updates and the patch no longer applies cleanly, the build dies rather than producing a subtly broken image.

### Feed Sources
- ImmortalWrt source: `https://github.com/padavanonly/immortalwrt-mt798x-24.10` branch `mt798x-mt799x-6.6-mtwifi`
- Config: `https://raw.githubusercontent.com/padavanonly/immortalwrt-mt798x-6.6/defconfig/mt7987_mt7992.config`
- Nikki feed: `nikkinikki-org/OpenWrt-nikki` (cloned on top of `feeds/nikki` if main nikki feed update fails)

### Download Mirrors
- Go proxy: `GOPROXY` (default: `https://goproxy.cn,https://proxy.golang.org,direct`)
- OpenWrt sources: `DOWNLOAD_MIRROR` (default: Tsinghua/USTC/BFSU)
- GitHub raw: `GITHUB_PROXY_PREFIXES` (ghfast.top, gh-proxy.com, gh.llkk.cc) — original URL always tried first

## GitHub Actions

- Scheduled: weekly Sunday 16:00 UTC (Monday 00:00 Beijing time)
- Manual trigger: all `ENABLE_*` are boolean inputs; `publish_release` controls Artifact-only vs Release publish
- Coverage config tests and firmware build run in parallel — coverage does not block Release publishing
- Feeds update failure aborts the build (avoids shipping firmware missing plugins)
- Artifact includes `build.config` and `enabled-packages.txt` for verification

## Repository Layout
```
.
├── scripts/
│   ├── local-build.sh            # THE canonical build script (all paths call this)
│   ├── local-build.ps1           # WSL2 wrapper (Windows)
│   └── coverage-test.sh          # Config coverage profiles
├── patches/
│   └── mtwifi-apcli-active-only.patch  # Persistent WiFi fix
├── feeds.conf.default            # OpenWrt feeds configuration
├── h5000m.extra.config           # Kernel/config fragments appended after defconfig
├── .github/workflows/build-test.yml  # GitHub Actions (calls local-build.sh)
└── artifacts/                    # Build output (gitignored)
```
