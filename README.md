# Bootleggers Android 13 Samsung mobile-data compatibility series

**Local only. Application checked; unbuilt and runtime-untested.** These are format-patch-compatible, zero-commit mboxes; no source commit was created here. Do not apply the older `ROM_ONLY_NETD_DIRECT_CONNECT_RULE_PROPOSAL.patch` as well as this series.

## Findings and evidence

The working UL capture has kernel IPv4 `/8`, Android `/32`, a gateway default in the named `rmnet1` table, cellular VALIDATED, and nonzero RX/TX. UL's policy rules contain:

```text
30000: from all fwmark 0x0/0xffff uidrange 0-0 lookup main
```

The first patch ports the equivalent guarded rule from UL `ed974e22e89f704f13b8ed1940afb51d959f47c8`; the second supplies strictly required Bootleggers Soong wiring. The existing on7xelte `TARGET_NEEDS_NETD_DIRECT_CONNECT_RULE := true` remains unchanged. No gateway, interface prefix, RIL, proprietary Samsung vendor, kernel, or hardware edits are included. The code-64 native-network lifecycle failure is not the same error as kernel gateway-validation code 101 and is not independently repaired by this series.

## Checked public bases

- `system/netd`: AOSP `android-13.0.0_r43`, `2a381fcd7849d46c59608e48026dae31116915dc`, selected by public Bootleggers manifest `1e6d8243dbd38a79937149af3b036a508aeafb03`.
- `vendor/bootleggers`: BootleggersROM/vendor_bootleggers `tirimbino`, `97670fe36551d4c90ff08fc84d5f42c971d8283f`.
- `git apply --check --verbose` passed for both patches against the saved public base files; see `application_check.json` and `../public_bases/source_manifest.json`. No public base file was patched during that check.

The actual private-server revisions must be checked separately. Stop rather than reset a private tree or duplicate an existing equivalent workaround.

## Exact application commands — run only when you choose to apply

Run from the private server's ROM source root after transferring this local-only series there yourself. Adjust `PATCH_DIR` only if you saved it somewhere else. These commands create one ROM commit per repository; they were not run here.

```sh
set -eu
ROM_ROOT="$PWD"
PATCH_DIR="$HOME/on7xelte/data_fix_20261124/rom_patch_series"

test "$(git -C "$ROM_ROOT/system/netd" rev-parse HEAD)" = 2a381fcd7849d46c59608e48026dae31116915dc
test "$(git -C "$ROM_ROOT/vendor/bootleggers" rev-parse HEAD)" = 97670fe36551d4c90ff08fc84d5f42c971d8283f
test -z "$(git -C "$ROM_ROOT/system/netd" status --porcelain)"
test -z "$(git -C "$ROM_ROOT/vendor/bootleggers" status --porcelain)"

git -C "$ROM_ROOT/system/netd" config --local user.name NothingnessN
git -C "$ROM_ROOT/system/netd" config --local user.email NothingnessN@users.noreply.github.com
git -C "$ROM_ROOT/system/netd" config --local --get user.name
git -C "$ROM_ROOT/system/netd" config --local --get user.email
git -C "$ROM_ROOT/system/netd" apply --check "$PATCH_DIR/0001-netd-restore-guarded-connected-route-lookup.patch"
git -C "$ROM_ROOT/system/netd" am "$PATCH_DIR/0001-netd-restore-guarded-connected-route-lookup.patch"
git -C "$ROM_ROOT/system/netd" log -1 --format='%an <%ae>'
NETD_FIX=$(git -C "$ROM_ROOT/system/netd" rev-parse HEAD)

git -C "$ROM_ROOT/vendor/bootleggers" config --local user.name NothingnessN
git -C "$ROM_ROOT/vendor/bootleggers" config --local user.email NothingnessN@users.noreply.github.com
git -C "$ROM_ROOT/vendor/bootleggers" config --local --get user.name
git -C "$ROM_ROOT/vendor/bootleggers" config --local --get user.email
git -C "$ROM_ROOT/vendor/bootleggers" apply --check "$PATCH_DIR/0002-bootleggers-wire-netd-direct-connect-flag.patch"
git -C "$ROM_ROOT/vendor/bootleggers" am "$PATCH_DIR/0002-bootleggers-wire-netd-direct-connect-flag.patch"
git -C "$ROM_ROOT/vendor/bootleggers" log -1 --format='%an <%ae>'
BOOTLEGGERS_VENDOR_FIX=$(git -C "$ROM_ROOT/vendor/bootleggers" rev-parse HEAD)
printf 'NETD_FIX=%s\nBOOTLEGGERS_VENDOR_FIX=%s\n' "$NETD_FIX" "$BOOTLEGGERS_VENDOR_FIX"
```

Record the two printed commit IDs for rollback. If either HEAD differs from the checked bases, these strict guards stop; review that revision before trying to apply. If `git am` starts but fails, abort only that repository's attempted application with `git -C "$ROM_ROOT/system/netd" am --abort` or `git -C "$ROM_ROOT/vendor/bootleggers" am --abort`, as appropriate. No push is part of these instructions.

## Build commands — not executed here

In your normal Bootleggers build shell after its existing lunch target has been selected:

```sh
m netd
m systemimage
```

`netd` rebuilds the executable and its `libnetd_server` dependency with the guarded rule. `systemimage` packages it; the ROM build-wiring edit does not require a proprietary Samsung vendor image rebuild. Use your normal ROM packaging process if it also regenerates super/OTA images. No lunch target, flash command, or on-device binary replacement is guessed here.

## Test checklist and read-only commands — for your later chosen build

First identify the active rmnet interface; replace `rmnet1` below if a reconnect selected another interface.

```sh
adb shell 'ip -o addr show | grep -E "^[0-9]+: rmnet[0-9]+"'
adb shell ip rule show
adb shell ip route show table rmnet1
adb shell ip -s link show dev rmnet1
adb shell 'dumpsys connectivity | grep -E "NetworkAgentInfo|VALIDATED"'
adb shell 'logcat -b radio -b main -b system -d -v threadtime | grep -E "SETUP_DATA_CALL|NIF.*(IPv4|mask)|Error adding route|Machine is not on the network|DataCallResponse"'
```

These raw terminal commands can show IPs/operator information; redact before sharing. The included evidence captures were already redacted.

- Browser: Wi-Fi off, intended default APN, multiple HTTPS sites load and DNS works.
- rmnet RX/TX: read counters before and after manual browser traffic; both advance.
- Routes: the named per-network table has its connected host route and gateway default; UID-0/netId-unset `main` lookup exists at priority 30000. Kernel `/8` with Android `/32` is acceptable for this workaround.
- Validation: cellular gains VALIDATED; distinguish any remaining code 64/native-network lifecycle failure from code 101/gateway-route failure within the same attempt.
- Hotspot: a client resolves DNS and loads HTTPS through the cellular upstream, without offload changes.
- Repeat reconnect and Wi-Fi/mobile handover; check ordinary calls, A2DP, and camera. No SCO/HFP work is included.

## Revert commands — preserve history

With clean ROM checkouts and the exact two commit IDs printed during application still assigned (or re-entered), revert only this series:

```sh
test -n "$BOOTLEGGERS_VENDOR_FIX"
test -n "$NETD_FIX"
git -C "$ROM_ROOT/vendor/bootleggers" config --local user.name NothingnessN
git -C "$ROM_ROOT/vendor/bootleggers" config --local user.email NothingnessN@users.noreply.github.com
git -C "$ROM_ROOT/vendor/bootleggers" config --local --get user.name
git -C "$ROM_ROOT/vendor/bootleggers" config --local --get user.email
git -C "$ROM_ROOT/vendor/bootleggers" revert "$BOOTLEGGERS_VENDOR_FIX"
git -C "$ROM_ROOT/vendor/bootleggers" log -1 --format='%an <%ae>'
git -C "$ROM_ROOT/system/netd" config --local user.name NothingnessN
git -C "$ROM_ROOT/system/netd" config --local user.email NothingnessN@users.noreply.github.com
git -C "$ROM_ROOT/system/netd" config --local --get user.name
git -C "$ROM_ROOT/system/netd" config --local --get user.email
git -C "$ROM_ROOT/system/netd" revert "$NETD_FIX"
git -C "$ROM_ROOT/system/netd" log -1 --format='%an <%ae>'
m netd
m systemimage
```

Do not build between the two repository reverts. No reset, force-push, protected device-branch modification, build, flash, or phone change was performed by the agent.
