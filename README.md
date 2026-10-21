# FocalTech 2808:a658 Fingerprint Driver for Arch Linux

Get the FocalTech / Realtek fingerprint reader (`2808:a658`, "Realtek USB2.0 Finger Print Bridge FocalTech Fingerprint Device", e.g. ASUS Vivobook Pro 15) working on **current Arch Linux**.

> **Fork notice:** this is a fork of
> [Meetsuthar32778/2808-A658-fingerprint-arch-linux-driver](https://github.com/Meetsuthar32778/2808-A658-fingerprint-arch-linux-driver).
> The original instructions no longer work on an up-to-date Arch (libgusb 0.4.x). This fork documents the cause and adds a fix.
>
> Ubuntu/Debian script inherited from the original repo, untested in this fork.
---

## The problem with the original instructions

The driver is a **proprietary, precompiled `libfprint`** (originally from the `ftfpteams` Debian/RPM package). It was linked against an old `libgusb` and requests symbols with the version tag `LIBGUSB_0.1.0`.

Current Arch ships `libgusb` 0.4.x, which exports the same functions under `LIBGUSB_0.2.8`. As a result `fprintd` exits immediately after start, and the GUI shows **"Remote peer disconnected"** when you try to enroll a finger:

```
fprintd: symbol lookup error: /usr/lib/libfprint-2.so.2:
undefined symbol: g_usb_device_get_interfaces, version LIBGUSB_0.1.0
fprintd.service: Main process exited, code=exited, status=127/n/a
```

Affected symbols:

- `g_usb_device_get_interfaces`
- `g_usb_device_get_release`
- `g_usb_interface_get_class`
- `g_usb_interface_get_subclass`
- `g_usb_interface_get_protocol`
- `g_usb_interface_get_number`

**Fix:** remove the version requirement on these symbols with `patchelf --clear-symbol-version`, so the dynamic linker resolves them against the installed `libgusb`.

Tested on: Arch Linux, ASUS Vivobook Pro 15, `libgusb 0.4.9`, `libfprint 1.94.100`, `patchelf 0.19.2`.

---

## Compatible hardware

| Device                       | USB ID      | Status                                   |
| ---------------------------- | ----------- | ---------------------------------------- |
| ASUS laptops (FocalTech/RTS) | `2808:a658` | Working with the patch below             |
| Other FocalTech readers      | `2808:xxxx` | Unknown. Different drivers are needed    |

Check your device:

```bash
lsusb | grep -i -E "2808|focaltech"
```

---

## Installation (Arch Linux)

### 1. Install dependencies and clone

```bash
sudo pacman -S --needed fprintd patchelf git
git clone https://github.com/<your-user>/2808-A658-fingerprint-arch-linux-driver.git
cd 2808-A658-fingerprint-arch-linux-driver
```

### 2. Extract the driver and patch it

```bash
rm -rf /tmp/fp && mkdir -p /tmp/fp
tar -xf libfprint.pkg.tar.zst -C /tmp/fp

F=/tmp/fp/usr/lib/x86_64-linux-gnu/libfprint-2.so.2.0.0

for s in g_usb_device_get_interfaces g_usb_device_get_release \
         g_usb_interface_get_class g_usb_interface_get_subclass \
         g_usb_interface_get_protocol g_usb_interface_get_number; do
  patchelf --clear-symbol-version "$s" "$F"
done

# Must print nothing:
ldd -r "$F" 2>&1 | grep -i -E "undefined|not found"
```

If the last command prints other missing symbols or libraries, the driver has additional incompatibilities. Open an issue with the output.

### 3. Keep a copy of the patched library

`/tmp` is wiped on reboot, so store the patched file somewhere permanent:

```bash
sudo install -Dm755 "$F" /usr/local/share/focaltech-a658/libfprint-2.so.2.0.0
```

### 4. Replace the system library

Back up the official one first, **outside of `/usr/lib`**:

```bash
sudo cp -a /usr/lib/libfprint-2.so.2.0.0 /usr/local/share/focaltech-a658/libfprint-2.so.2.0.0.official
sudo cp /usr/local/share/focaltech-a658/libfprint-2.so.2.0.0 /usr/lib/libfprint-2.so.2.0.0
sudo systemctl restart fprintd
```

> **Important:** never leave a backup such as `libfprint-2.so.2.0.0.official` inside `/usr/lib`. `ldconfig` sorts library file names and would re-point the `libfprint-2.so.2` symlink to the backup (`libfprint-2.so.2.0.0.official` sorts after `libfprint-2.so.2.0.0`), so `fprintd` silently loads the official library and reports "No devices available".

### 5. Verify and enroll

```bash
fprintd-list "$USER"     # should list "FocalTech Systems Co., Ltd fingerprint"
fprintd-enroll           # touch the sensor repeatedly until enroll-completed
fprintd-verify
```

Then you can also enroll from your desktop settings (GNOME: Settings -> System -> Users -> Fingerprint Login).

> **Do not** use the old step `pacman -U -dd --overwrite '*' libfprint.pkg.tar.zst`. It replaces the system `libfprint` with a package built for Debian and conflicts with the Arch package.

---

## After a `libfprint` update: re-apply the patch

Whenever `pacman` updates `libfprint`, the official library (without the FocalTech driver) overwrites the patched one and the reader stops working (`fprintd-list` finds no device, or the fingerprint option disappears from the settings).

You do **not** need to patch anything again: the patched driver library is the one saved in step 3. Just put it back:

```bash
sudo cp /usr/local/share/focaltech-a658/libfprint-2.so.2.0.0 /usr/lib/libfprint-2.so.2.0.0
sudo systemctl restart fprintd
fprintd-list "$USER"
```

Check that the right library is in place:

```bash
nm -D /usr/lib/libfprint-2.so.2.0.0 | grep get_interfaces
# must NOT show "@LIBGUSB_0.1.0"
```

Your enrolled fingerprints are kept in `/var/lib/fprint/` and are not affected, so you do not need to enroll again.

If the saved copy is lost, repeat steps 2 and 3 to extract and patch the driver again from `libfprint.pkg.tar.zst`.

Things to know:

- The old driver may stop working after future updates of `fprintd`, `libgusb`, `glib` or `openssl`. If restoring the library is not enough, check `journalctl -u fprintd -b --no-pager | tail -30` and open an issue.
- Optional: if you prefer to avoid doing this after every update, you can add `IgnorePkg = libfprint` to the `[options]` section of `/etc/pacman.conf`. The downside is that `libfprint` will no longer receive updates, including security fixes.

---

## Optional: fingerprint for `sudo`

Add this line at the **top** of `/etc/pam.d/sudo`:

```
auth    sufficient    pam_fprintd.so
```

Warnings:

- `sufficient` means the fingerprint alone authenticates `sudo`. Per the Arch Wiki, this allows a background process to trigger `sudo` while you touch the sensor for another reason.
- Before editing PAM files, keep a root shell open (`sudo -s`) and test in a second terminal, so a mistake does not lock you out.
- Safer alternative: `pam-fprint-grosshack` (AUR), which accepts the password and the fingerprint at the same prompt. Review the PKGBUILD before installing.

---

## Troubleshooting

**"Remote peer disconnected" when enrolling**

`fprintd` is crashing at startup. Check:

```bash
journalctl -u fprintd -b --no-pager | tail -30
```

If you see `undefined symbol ... LIBGUSB_0.1.0`, repeat the patch above.

**Everything stopped working after an update or a reboot**

The official `libfprint` has probably overwritten the patched one:

```bash
pacman -Qo /usr/lib/libfprint-2.so.2
nm -D /usr/lib/libfprint-2.so.2.0.0 | grep get_interfaces
grep -i libfprint /var/log/pacman.log | tail
```

Re-copy the patched library as described in the section "After a `libfprint` update".

**`fprintd-list` says "No devices available" although `lsusb` shows `2808:a658`**

Check where the `libfprint-2.so.2` symlink points:

```bash
ls -l /usr/lib/libfprint-2.so*
```

It must point to `libfprint-2.so.2.0.0`. If it points to something else (for example a `.official` backup), move the backup out of `/usr/lib` and fix the link:

```bash
sudo mv /usr/lib/libfprint-2.so.2.0.0.official /usr/local/share/focaltech-a658/
sudo ln -sf libfprint-2.so.2.0.0 /usr/lib/libfprint-2.so.2
sudo ldconfig
sudo systemctl restart fprintd
```

**`fprintd-list` says no devices**

- Check that `lsusb` shows `2808:a658`.
- Check that the library in `/usr/lib` is the patched one (see above).
- Run `sudo udevadm control --reload && sudo udevadm trigger`, or reboot.
- Some laptops need the reader enabled in BIOS/UEFI.

**Enrolled fingers disappeared**

Fingerprints are stored in `/var/lib/fprint/`. If that directory was deleted or the device ID changed, enroll again.

**Go back to the official library**

```bash
sudo cp -a /usr/local/share/focaltech-a658/libfprint-2.so.2.0.0.official /usr/lib/libfprint-2.so.2.0.0
# or:
sudo pacman -S --overwrite '*' libfprint
sudo systemctl restart fprintd
```

---

## Notes and limitations

- The driver is **proprietary and closed source** (from `ftfpteams`). This repository only redistributes the original binary package. The patch is applied locally on your machine and the modified binary is not redistributed. Check the original license terms before sharing modified binaries.
- Because the binary is old, other future changes in `glib`, `libusb`, `openssl`, or `fprintd` may break it again.
- Upstream `libfprint` has no official driver for this reader.

## Credits

- Original repository: [Meetsuthar32778](https://github.com/Meetsuthar32778/2808-A658-fingerprint-arch-linux-driver)
- Driver: `ftfpteams` (FocalTech / Realtek RTS5811 FT9366)
- Libgusb symbol fix: documented in this fork

## Contributing

Tested on another model or distro? Open an issue or pull request with your `lsusb` output, `pacman -Q libgusb libfprint fprintd`, and what worked.
