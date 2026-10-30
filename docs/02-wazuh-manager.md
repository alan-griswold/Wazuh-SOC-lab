# Deploying a Wazuh Manager VM on Proxmox

**One-line summary:** Built an Ubuntu Server VM from the CLI on my Proxmox host and installed an all-in-one Wazuh SIEM (manager, indexer, dashboard) on it, working through a boot-order loop, a half-allocated LVM disk, and a leftover failed installation.

## Goal

Stand up the SIEM at the center of my SOC lab so isolated lab VMs can send it logs. The VM needed two network connections: one on the management network so I can reach the dashboard and fetch updates, and one on the isolated lab bridge where monitored machines will report. I also wanted to build it from the command line (`qm`) instead of the web UI, to get comfortable with Linux and Proxmox's CLI.

## Architecture

- **VM 100 (`wazuh`)** on Proxmox VE 9.1: 4 vCPUs, 8 GiB RAM, 60 GiB thin-provisioned disk on `local-lvm`, VirtIO NICs and disk controller, QEMU guest agent enabled.
- **OS:** Ubuntu Server 24.04.5 LTS.
- **Software:** Wazuh 4.14.9, installed all-in-one (manager, indexer, dashboard) with Wazuh's install assistant.
- **Two NICs:**

| Interface | Bridge | Address | Purpose |
|---|---|---|---|
| `ens18` | `vmbr0` (management) | `192.168.8.185/24` (DHCP) | Dashboard access, updates |
| `ens19` | `vmbr1` (isolated lab) | `10.10.10.10/24` (static, no gateway) | Agents in the lab report here |

- **Why no gateway on `ens19`:** `vmbr1` has no uplink, so nothing on it leads anywhere. A second gateway would also compete with the default route through `ens18`.
- **Why `10.10.10.0/24`:** a private range that can't collide with the Beryl's `192.168.8.0/24`, and it makes lab addresses easy to spot in logs.

Dashboard access path: laptop (any network) → Tailscale → SSH to the Proxmox host → forwarded to the VM's dashboard on port 443.

## Steps Taken

1. Fixed the Proxmox apt repositories (disabled the enterprise repos, confirmed `pve-no-subscription`) and upgraded the host to kernel 7.0.14-26, after making `nomodeset` permanent in `/etc/default/grub` and running `update-grub` so a reboot wouldn't hang on the Nvidia driver.
2. Downloaded the Ubuntu Server ISO into `/var/lib/vz/template/iso/` and verified it with `sha256sum`.
3. Planned a RAM budget for the lab (14 GiB visible on the host) and chose VM sizes.
4. Created the VM with `qm create` (see Smaller Mistakes).
5. Installed Ubuntu through the VM console, configuring `ens19` manually with no gateway.
6. Fixed the boot order and ejected the ISO (Problem 1).
7. Updated the system and installed `qemu-guest-agent`, then verified with `qm agent 100 ping`.
8. Took a `clean-ubuntu` snapshot before installing anything else.
9. Downloaded and ran the Wazuh installer (Smaller Mistakes, Problems 2 and 3).
10. Took a `wazuh-installed` snapshot and reached the dashboard through an SSH tunnel.

## Problem 1: Installer Appeared Again After the Install

**Symptom:** After the install finished, the VM booted back into the Ubuntu installer. Thinking the install had failed, I ran it a second time and got the same result, along with `[FAILED] Failed unmounting cdrom.mount - /cdrom`.

**Diagnosis:** The `FAILED` line was cosmetic: the installer was asking me to remove the install media, and the ISO was still in the VM's virtual CD drive. The install itself had actually succeeded the first time. I had set the boot order to CD first (`--boot order='ide2;scsi0'`), so every reboot went to the ISO.

**Root cause:** Boot order put the CD ahead of the disk, and the ISO was never ejected.

**Fix:** Ejected the ISO with `qm set 100 --ide2 none,media=cdrom` and changed the boot order to disk only with `qm set 100 --boot order=scsi0`, then checked with `qm config 100`. My first attempt had a typo (`non`), and the error `unable to parse volume ID 'non'` pointed straight at it.

**Lesson:** When a result looks wrong, find out what the system is actually doing before assuming the previous step failed. Also, set the boot order back as soon as the install is done.

## Problem 2: Dashboard Install Failed, `No space left on device`

**Symptom:** The installer got through the indexer, manager, and Filebeat, then failed at the dashboard step and rolled everything back.

**Diagnosis:** Searched the log with `sudo grep -n -i -E "error|fail" /var/log/wazuh-install.log` and found `Write error - write (28: No space left on device)` and `dpkg-deb: ... disk full`. `df -h /` afterward showed 32% used, which looked fine, but that was because the rollback had freed the space. `lsblk` and `sudo vgs` showed the real picture: the disk was 60 GB, but the logical volume holding `/` was only 29 GB, with 29 GB sitting unallocated in the volume group (`VFree`).

**Root cause:** Ubuntu's installer default left half the LVM volume group unallocated.

**Fix:** `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` grew the volume and its filesystem online. `df -h /` then showed 57 GB.

**Lesson:** `df` shows the disk right now, not when it failed. The layers (physical disk, volume group, logical volume, filesystem) are separate, and `lsblk`, `vgs`, and `lvs` show where space is going.

## Problem 3: Leftover Manager Blocked the Reinstall

**Symptom:** After the failure, `dpkg -l | grep -i wazuh` still listed `wazuh-manager` as installed (`ii`). Rerunning the installer said `Wazuh manager already installed`. Adding `-o` (overwrite) produced `Port 1515 is being used` and `Port 55000 is being used`, which are the manager's enrollment and API ports.

**Root cause:** The first failed run left a running `wazuh-manager` that Wazuh's own cleanup couldn't remove ("package could not be removed"). It held the ports, so the installer refused to continue.

**Fix:** Rolled back to the clean-ubuntu snapshot, which also reverted my disk extension, so I re-ran lvextend and then reran the installer, which completed.

**Lesson:** A failed install can leave a half-installed state that causes confusing follow-up errors. Snapshotting before a risky install gives you a clean way back.

## Smaller Mistakes

- **`qm create` returned `400 too many arguments`.** I had typed `-- net1` with a space, which makes everything after a bare `--` a plain argument, and pasted line-continuation backslashes ended up mid-line. Fixed by running it as one line. Lesson: "too many arguments" means something broke into separate words, so read the command piece by piece.
- **The Wazuh installer failed with `syntax error near unexpected token 'newline'`.** The download URL had `4,14` instead of `4.14`, so the server returned an XML error page that `curl -s` hid and `bash` then tried to run as a script. Fixed the URL and switched to `curl -fsSLO`, where `-f` fails on server errors. Lesson: look at what you downloaded before running it as root. Line 1 would have shown right away that it wasn't a script.

## Result / Verification

- The installer finished with `Installation finished`, and all three services (`wazuh-manager`, `wazuh-indexer`, `wazuh-dashboard`) are running.
- The dashboard loads at `https://localhost:8443` through an SSH tunnel to the Proxmox host:
  ```
  ssh -L 8443:192.168.8.185:443 root@<proxmox-tailscale-ip>
  ```
  The `-L <local-port>:<destination>:<destination-port>` flag makes the Proxmox host forward traffic to the VM's dashboard, so it's one port on one machine rather than a whole network.
- Took a `wazuh-installed` snapshot as a restore point.

## What's Next

- Change the generated admin password and store it in a password manager.
- Reserve `192.168.8.185` on the Beryl for the VM's MAC, since it's DHCP and could change.
- Build a small Ubuntu VM on `vmbr1` only, install a Wazuh agent pointing at `10.10.10.10`, and confirm logs arrive.
- Use `qm clone` and templates so I don't repeat the installer for every VM.
- Decide how the Raspberry Pi honeypot will report in without sitting on the same LAN as my real devices.

---

*Lessons for next time: (1) snapshot before any risky install; (2) check how much space the installer actually gave `/`, not how much the disk has; (3) read what you downloaded before running it; (4) set the boot order back when the OS is installed.*
