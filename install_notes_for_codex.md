Install Notes for Codex
=======================

Purpose
-------

This file documents the exact recovery and installation path that actually worked for installing stock `openpilot 0.8.13.1` on a `LeEco Le Pro 3 X720` (`LEX720`, `le_zl1`).

This is not a generic Android flashing guide. It is a practical note for future recovery work on this specific legacy device.


What Actually Mattered
----------------------

The critical finding was that the first NEOS base used during earlier attempts was too old:

- `/VERSION = 10`
- `Python 3.7.3`

That base led to repeated runtime incompatibilities with `openpilot 0.8.13.1`.

The working base was:

- `NEOS 20`
- `/VERSION = 20`
- `Python 3.8.5`

Once `NEOS 20` was installed, `openpilot 0.8.13.1` could be deployed cleanly without the earlier Python 3.7 compatibility failures.


Required Files
--------------

For NEOS recovery:

- `ota-signed-e5aa34ebb27977779db4e82439cca8f807e9c9ee2c84c217c926a2d08dd2959f.zip`
- `recovery-cf752ceb7931aa427ccec384fd21d9425c6053f25161814dab31ec75e4d296b0.img`

These are the official `NEOS 20` assets.

For openpilot:

- a complete checkout of `commaai/openpilot` branch `commatwo_master`
- all submodules initialized recursively

The branch version must resolve to:

- `COMMA_VERSION "0.8.13.1"`


Device Identity Checks
----------------------

Expected values on the LeEco device:

- `ro.product.model = LEX720`
- `ro.product.device = le_zl1`

Useful commands:

```bash
adb devices
adb shell getprop ro.product.model
adb shell getprop ro.product.device
```


Bootloader Requirements
-----------------------

Bootloader must already be unlocked.

Useful commands:

```bash
adb reboot bootloader
fastboot devices
fastboot oem device-info
```

Expected:

- `Device unlocked: true`
- `Device critical unlocked: true`


NEOS 20 Flash Procedure
-----------------------

1. Extract `boot.img` and `system.img` from the OTA zip.
2. Reboot the phone into fastboot.
3. Flash recovery, boot, and system.
4. Erase userdata and cache.
5. Reboot.

Example:

```bash
unzip -o ota-signed-e5aa34ebb27977779db4e82439cca8f807e9c9ee2c84c217c926a2d08dd2959f.zip 'files/*'

fastboot flash recovery recovery-cf752ceb7931aa427ccec384fd21d9425c6053f25161814dab31ec75e4d296b0.img
fastboot flash boot files/boot.img
fastboot flash system files/system.img
fastboot erase userdata
fastboot erase cache
fastboot reboot
```

Notes:

- `fastboot format cache` may fail on modern host tools.
- `fastboot erase cache` was sufficient.
- If `fastboot flash` hangs on `< waiting for any device >`, kill stale `fastboot` processes first and retry.


Expected State After Correct NEOS Flash
---------------------------------------

After first boot and Wi-Fi setup, the device should expose:

- `/VERSION = 20`
- `Python 3.8.5`

Useful command after SSH access:

```bash
cat /VERSION
python3 --version
```


Important NEOS Setup Behavior
-----------------------------

The `NEOS Setup` UI may say that Wi-Fi has "no internet" even when network access is actually working.

Do not trust the on-screen connectivity warning by itself.

What to do instead:

1. Connect the phone to Wi-Fi anyway.
2. Open network details.
3. Get the IPv4 address.
4. Test reachability from the PC.

Example:

```bash
ping -c 3 <phone-ip>
ssh -p 8022 -i id_rsa root@<phone-ip>
```


SSH Access During Setup
-----------------------

During setup, the device exposed SSH on port `8022`.

That was enough to bypass the unreliable setup downloader and install `openpilot` manually.

Useful command:

```bash
ssh -p 8022 -i id_rsa root@<phone-ip>
```


openpilot 0.8.13.1 Install Procedure
------------------------------------

1. Clone `commaai/openpilot` branch `commatwo_master`.
2. Initialize submodules recursively.
3. Confirm version header is `0.8.13.1`.
4. Create a tarball without `.git`.
5. Copy the tarball to the device over SSH.
6. Extract it into `/data/openpilot`.
7. Create `/data/data/com.termux/files/continue.sh`.
8. Reboot.

Example on host:

```bash
git clone https://github.com/commaai/openpilot.git -b commatwo_master /tmp/op-c2-check
cd /tmp/op-c2-check
git submodule update --init --recursive
sed -n '1,3p' selfdrive/common/version.h
tar --exclude=.git -czf /tmp/openpilot-c2-08131-full.tgz .
scp -P 8022 -i id_rsa /tmp/openpilot-c2-08131-full.tgz root@<phone-ip>:/data/openpilot-c2-08131-full.tgz
```

Example on device:

```bash
rm -rf /data/openpilot
mkdir -p /data/openpilot /data/data/com.termux/files
cd /data/openpilot
tar xzf /data/openpilot-c2-08131-full.tgz
rm -f /data/openpilot-c2-08131-full.tgz

cat > /data/data/com.termux/files/continue.sh <<'EOF'
#!/usr/bin/bash
cd /data/openpilot
exec ./launch_openpilot.sh
EOF

chmod +x /data/data/com.termux/files/continue.sh
reboot
```


Why Earlier Attempts Failed
---------------------------

These were the real failure causes during earlier attempts:

- wrong NEOS base (`VERSION 10`, Python 3.7.3)
- relying on the broken NEOS setup downloader
- trying newer forks on an old runtime
- trying to use an incomplete `openpilot` checkout without submodules
- spending time on Python dependency patching that became unnecessary once `NEOS 20` was installed


What Did Not Matter In The Final Working Path
---------------------------------------------

These were dead ends or non-essential:

- forcing `openpilot.comma.ai` through the setup UI
- using `OPKR` for this specific target
- patch-only community `update.zip` files for `OnePlus 3T`
- trying to make `Python 3.7` work with `openpilot 0.8.13.1`
- compiling `numpy` on the old base


Recommended Disaster Recovery Bundle
------------------------------------

For future reinstalls, keep all of the following together:

1. `NEOS 20` release assets
   - OTA zip
   - recovery image
2. Extracted `files/boot.img`
3. Extracted `files/system.img`
4. Full `openpilot 0.8.13.1` archive with submodules included
5. Setup SSH key used for NEOS setup access
6. This note file


Minimal Future Recovery Flow
----------------------------

If this device needs to be rebuilt again, the shortest reliable path is:

1. Verify `LEX720 / le_zl1`
2. Enter fastboot
3. Flash official `NEOS 20`
4. Join Wi-Fi
5. Ignore the setup internet warning if the device is reachable from the PC
6. SSH to port `8022`
7. Copy full `openpilot 0.8.13.1`
8. Install to `/data/openpilot`
9. Create `continue.sh`
10. Reboot

