# ONE-SHOT AGENT PROMPT: Create macOS Tahoe VM on Proxmox 9

**GOAL:** Create a macOS Tahoe virtual machine on a Proxmox VE 9 host and boot it to the macOS Recovery installer screen. No Mac hardware is required. Do not use the osx-proxmox GitHub repo — it no longer works reliably.

---

### ENVIRONMENT FACTS

* Proxmox VE version: **9.x** (Proxmox 9 has stricter config validation than 8.x — some options valid in 8.x will cause parse errors in 9)
* Host CPU: Modern Intel or AMD CPU with AVX2 support — emulate as **Haswell** for macOS compatibility
* Storage Pool Name: `<your-storage-pool>` (e.g., `local-zfs`, `local-lvm`, `ceph-rbd`) — must support raw or qcow2 formats
* ISO storage path: `/var/lib/vz/template/iso/`
* Proxmox web UI: `https://<your-proxmox-ip>:8006`
* VM ID to create: `<vmid>` (e.g., **101**)

---

### RESOURCES / REFERENCE LINKS

* OpenCore bootloader releases (use **v21**): [https://github.com/thenickdude/KVM-Opencore/releases/tag/v21](https://github.com/thenickdude/KVM-Opencore/releases/tag/v21)
* Nick Sherlock's Proxmox macOS guide (base reference, though some details are for older Proxmox — see critical notes below): [https://www.nicksherlock.com/2022/10/installing-macos-13-ventura-on-proxmox/](https://www.nicksherlock.com/2022/10/installing-macos-13-ventura-on-proxmox/)
* macOS fetch script (supports Tahoe): [https://github.com/kholia/OSX-KVM/blob/master/fetch-macOS-v2.py](https://github.com/kholia/OSX-KVM/blob/master/fetch-macOS-v2.py) — use board ID `Mac-CFF7D910A743CAAF` for Tahoe
* OpenCore KVM repo (reference for config): [https://github.com/AngeloAvv/OSX-KVM/tree/tahoe-support](https://github.com/AngeloAvv/OSX-KVM/tree/tahoe-support)

---

### CRITICAL LESSONS LEARNED (read before starting)

1. **OpenCore-v21.iso and the Tahoe recovery image are NOT ISO 9660 files.** They are GPT-partitioned raw disk images. The `file` command will show them as "DOS/MBR boot sector" and `isoinfo` will say "CD-ROM is NOT in ISO 9660 format". **You MUST attach them as SATA disks (`media=disk`), NOT as CD-ROMs.** If attached with `media=cdrom`, OVMF/UEFI will fail to boot them and fall through to a network PXE loop.
2. **`media=disc` is invalid in Proxmox 9.** Only `cdrom` and `disk` are valid values for the `media` parameter. Proxmox 9 will refuse to start the VM if any config line uses the legacy typo `media=disc`.
3. **Import disk images using `qm importdisk`, not by attaching them as ISOs.** After copying images to `/var/lib/vz/template/iso/`, use `qm importdisk <vmid> <file> <your-storage-pool> --format raw` to import each one as a proper disk. Attach the resulting disks as `sata0` (OpenCore) and `sata2` (Tahoe recovery) with `cache=unsafe`.
4. **The OSK key is required** in the QEMU args. The value is: `ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc`
5. **EFI disk size**: Use `efitype=4m,size=1M` — Proxmox will provision the 4MB OVMF vars file automatically.
6. **kvm ignore_msrs must be enabled** or macOS will kernel panic on boot. Set it both at runtime and persistently on the host.
7. **Do not use VNC console immediately on boot** — the OpenCore picker appears within a few seconds. If you miss it, the VM may attempt to boot other unconfigured devices.

---

### STEP-BY-STEP INSTRUCTIONS

#### STEP 1 — Download OpenCore v21

In the Proxmox host shell (`pve` node → Shell):

```bash
cd /var/lib/vz/template/iso/
wget https://github.com/thenickdude/KVM-Opencore/releases/download/v21/OpenCore-v21.iso.gz
gunzip OpenCore-v21.iso.gz
# Result: OpenCore-v21.iso (~150 MB) — note: this is a raw disk image, NOT a real ISO

```

#### STEP 2 — Download the macOS Tahoe Recovery Image

```bash
cd /tmp
wget https://raw.githubusercontent.com/kholia/OSX-KVM/master/fetch-macOS-v2.py
python3 fetch-macOS-v2.py --board-id Mac-CFF7D910A743CAAF --os latest
# This downloads BaseSystem.dmg from Apple's CDN (~900 MB)

# Convert to raw disk image
qemu-img convert -f dmg -O raw BaseSystem.dmg /var/lib/vz/template/iso/Tahoe-recovery.iso
# Result: Tahoe-recovery.iso (~2.5 GB) — also a raw disk image, NOT a real ISO

```

#### STEP 3 — Enable KVM ignore_msrs (Required Host Tweak)

```bash
echo "options kvm ignore_msrs=1 report_ignored_msrs=0" > /etc/modprobe.d/kvm.conf
update-initramfs -u -k all
# Apply immediately without rebooting the host:
echo 1 > /sys/module/kvm/parameters/ignore_msrs

```

#### STEP 4 — Create the VM via Proxmox UI

Navigate to the Proxmox web UI and use **Create VM** with these settings (replace `<vmid>` and `<your-storage-pool>` with your environment specifics):

| Tab | Setting | Value |
| --- | --- | --- |
| General | VM ID | `<vmid>` (e.g., 101) |
| General | Name | `macOS-Tahoe` |
| OS | ISO image | (Do not select an ISO / Do not use any media) |
| OS | Guest OS Type | Other |
| System | Machine | q35 |
| System | BIOS | OVMF (UEFI) |
| System | EFI Storage | `<your-storage-pool>` |
| System | EFI pre-enroll keys | **unchecked/no** |
| System | QEMU Agent | enabled |
| Disks | Bus/Device | VirtIO Block (0) |
| Disks | Storage | `<your-storage-pool>` |
| Disks | Size | 120 GB (or preferred size) |
| Disks | Cache | Write back |
| Disks | Discard | enabled |
| Disks | IO Thread | enabled |
| CPU | Type | Haswell |
| CPU | Cores | 4 (or more depending on host capacity) |
| CPU | Sockets | 1 |
| Memory | RAM | 4096 MB (Minimum recommended) |
| Memory | Ballooning | **disabled** |
| Network | Model | VirtIO (paravirt) |
| Network | Bridge | vmbr0 |

After VM creation, go to **Hardware → Display** and change it to **VMware compatible** (select from dropdown, do not type it).

#### STEP 5 — Import both disk images as SATA disks

Run these commands in the Proxmox host shell, substituting your target `<vmid>` and `<your-storage-pool>`.

```bash
# Set your variables here for easy execution copy-paste:
VMID=101
STORAGE="local-zfs"

# Import OpenCore as a disk
qm importdisk $VMID /var/lib/vz/template/iso/OpenCore-v21.iso $STORAGE --format raw

# Import Tahoe recovery as a disk (takes 30-60 seconds)
qm importdisk $VMID /var/lib/vz/template/iso/Tahoe-recovery.iso $STORAGE --format raw

```

#### STEP 6 — Automatically Inject QEMU Args and Attach Disks

Stop the VM first if running: `qm stop <vmid>`

Run this automated Python helper script in your shell. It will parse your newly imported unattached disks, clear conflicting IDE placeholders, inject the required OSK key arguments, and bind the images to the correct bootable SATA buses.

```bash
python3 - << 'PYEOF'
import re, sys

# --- CONFIGURATION ---
vmid = "101"  # Change to your target VM ID
# ---------------------

conf_path = f'/etc/pve/qemu-server/{vmid}.conf'

try:
    with open(conf_path, 'r') as f:
        lines = f.readlines()
except FileNotFoundError:
    print(f"Error: Config file for VM {vmid} not found.")
    sys.exit(1)

new_lines = []
unused_disks = []

# Scan for newly imported unused disks
for line in lines:
    s = line.strip()
    if s.startswith('unused'):
        match = re.search(r'unused\d+:\s+(.+)', s)
        if match:
            unused_disks.append(match.group(1))

if len(unused_disks) < 2:
    print("Warning: Could not find at least two unused disks. Ensure step 5 ran successfully.")

opencore_disk = next((d for d in unused_disks if "disk-2" in d or "OpenCore" in d), unused_disks[0] if unused_disks else None)
tahoe_disk = next((d for d in unused_disks if "disk-3" in d or "recovery" in d), unused_disks[1] if len(unused_disks) > 1 else None)

for line in lines:
    s = line.strip()
    # Clean up conflicting items
    if any(s.startswith(k) for k in ['ide0:', 'ide1:', 'ide2:', 'unused0:', 'unused1:', 'args:', 'boot:']):
        continue
    new_lines.append(line)

# Add standardized configurations
new_lines.append('boot: order=sata0;sata2;virtio0;net0\n')

osk = 'ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc'
args = (
    f'-device isa-applesmc,osk="{osk}" '
    f'-smbios type=2 '
    f'-device usb-kbd,bus=ehci.0,port=2 '
    f'-cpu Haswell,kvm=on,vendor=GenuineIntel,'
    f'+kvm_pv_unhalt,+kvm_pv_eoi,vmware-cpuid-freq=on,+invtsc'
)
new_lines.append(f'args: {args}\n')

if opencore_disk:
    new_lines.append(f'sata0: {opencore_disk},cache=unsafe,size=150M\n')
    print(f"Mapping OpenCore bootloader to sata0: {opencore_disk}")
if tahoe_disk:
    new_lines.append(f'sata2: {tahoe_disk},cache=unsafe,size=2542M\n')
    print(f"Mapping Tahoe Recovery to sata2: {tahoe_disk}")

with open(conf_path, 'w') as f:
    f.writelines(new_lines)

print(f'Config for VM {vmid} updated successfully.')
PYEOF

```

#### STEP 7 — Start the VM and open the Console

```bash
qm start <vmid>

```

Navigate to your Proxmox web UI → **VM `<vmid>**` → **Console**.

Within a few seconds, the **OpenCore boot picker** screen will populate with the following items:

* **macOS Base System** ← select this (it should be highlighted by default)
* UEFI Shell
* Reset NVRAM

Click inside the VNC web console wrapper and press **Enter**.

#### STEP 8 — Run the Installer Environment

The hardware initialization verbose log trail will roll by, followed by the Apple logo. Within 60 seconds, the **macOS Recovery** UI menu option block will load.

1. Open **Disk Utility** first → select the unformatted 120 GB VirtIO block storage disk → click **Erase**.
2. Format the layout explicitly as **APFS** and name the target volume (e.g., `Macintosh HD`). Close Disk Utility.
3. Select **Reinstall macOS Tahoe** → click Continue and target the newly created APFS volume.
4. The system execution loops will cycle and restart the VM multiple times during package extraction. During reboots, verify that OpenCore hands off tracking to **macOS Installer** initially, and then finally changes to target your custom name (`Macintosh HD`) to wrap up configuration.

---

### TROUBLESHOOTING

| Symptom | Cause | Fix |
| --- | --- | --- |
| VM stuck in PXE boot loop | Images attached as `media=cdrom` | Re-import images with `qm importdisk` and attach explicitly as raw SATA virtual disks. Do not use ISO mounters. |
| "value 'disc' does not have a value..." error | Proxmox 9 syntax validation failure | Ensure your `.conf` contains no references to legacy typo strings like `media=disc`. Proxmox 9 strictly accepts `cdrom` or `disk`. |
| OVMF shows "Failed to load Boot000X..." | Stale EFI boot NVRAM mapping vars | Select **Reset NVRAM** from the OpenCore picker list selection boot menu, or delete and recreate the `efidisk0` block asset. |
| macOS kernel panics immediately | Host CPU is dropping explicit instruction hooks | Execute runtime verification directly on host command line: `echo 1 > /sys/module/kvm/parameters/ignore_msrs` |
