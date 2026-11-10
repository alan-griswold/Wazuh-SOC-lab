# Installing Proxmox VE and Securing Remote Access (HP Pavilion Gaming Desktop)

Installed Proxmox VE as a bare-metal hypervisor on a second internal drive in my gaming PC (dual-booting with Windows), worked through an upstream installer bug, an Nvidia display conflict, and a network misconfiguration, then locked it down for key-only remote access over Tailscale.

## Goal

Stand up a dedicated virtualization host to build a home SOC/log-monitoring lab (Wazuh, detection rules, multiple VMs) plus isolated vulnerable machines for pen testing, as hands-on reinforcement for Security+ and a real portfolio project. It needed to run on real hardware (not a hosted-VM workaround), and I needed to manage it remotely from a dorm network I don't control.

## Architecture

- **HP Pavilion Gaming Desktop (TG01 series):** AMD CPU, GTX 1660 Ti, Samsung 870 EVO 1TB (Windows) plus a second, previously unused ~477GB internal drive discovered via Disk Management.
- **Proxmox VE 9.1** installed directly onto the second drive, dual-booting alongside Windows via UEFI boot selection (Esc, then F9 on this board). Proxmox is now the default boot entry.
- **GL.iNet Beryl travel router** bridging the dorm Wi-Fi to a wired connection for the Pavilion (its only NIC is Ethernet). Beryl LAN is `192.168.8.0/24`.
- **Two Proxmox bridges:** `vmbr0` (management, connected to the physical NIC) and `vmbr1` (isolated lab network with no physical port, reserved for Wazuh agents, Kali, and vulnerable VMs).
- **Tailscale** on the Proxmox host and my laptop for remote access without port forwarding.
- **4TB external HDD** reserved as future Proxmox storage for VM disks, ISOs, and backups.

Remote path: laptop (any network) → Tailscale tunnel → Proxmox host (`100.x` address) → SSH (key-only) or web UI on port 8006.

## Steps Taken

1. Found a second unused internal drive via Disk Management, which avoided shrinking the Windows partition.
2. Downloaded the Proxmox ISO, verified its SHA256, and flashed it with Rufus (DD mode, since Proxmox's hybrid ISO forces it).
3. Identified the boot-menu keys for this board (Esc, then F9) after an early attempt booted back into Windows from the wrong entry.
4. Installed to the ~477GB drive, confirming the target was not the Windows drive.
5. Fixed the network configuration after the install (Problem 4).
6. Created `vmbr1`, the isolated lab bridge.
7. Installed Tailscale on the host, joined it to my tailnet, and disabled key expiry for the node.
8. Set up key-based SSH, then disabled password authentication (Problem 6).
9. Set Proxmox as the default UEFI boot entry so a reboot or power loss doesn't leave the machine in Windows with no remote access.

## Problem 1: Boot Failure, `modprobe: FATAL: Module shpchp not found`

**Symptom:** The installer hung right after loading drivers, citing a missing kernel module.

**Diagnosis:**
- The SHA256 of the ISO matched Proxmox's published hash, which ruled out a corrupted download.
- Re-flashing with Rufus's bad-block check enabled finished with 0 errors, which ruled out a bad write.
- Re-flashing to a second, different drive gave the exact same error, which ruled out a drive-specific problem.
- Searching the exact error text turned up multiple reports on the Proxmox forum of this failure on the 9.2 ISO specifically, with 9.1 confirmed working.

**Root cause:** A packaging bug in the Proxmox VE 9.2 ISO itself (a kernel module missing from the installer's initramfs), not my download, media, or hardware.

**Fix:** Switched to the 9.1 ISO, verified its checksum, and re-flashed. It booted.

## Problem 2: Installer Hang (Nvidia Driver Conflict)

**Symptom:** After switching to 9.1, the installer stalled during driver loading, with `nvidiafb` visible right before the hang.

**Diagnosis:** Forum reports showed other users with Nvidia cards hitting display freezes on the Proxmox installer tied to the Nvidia framebuffer driver initializing at boot.

**Root cause:** The default kernel boot parameters let the Nvidia driver try to take over the display, which hung on this hardware.

**Fix:** At the installer's GRUB menu I pressed `e`, appended `nomodeset` to the end of the existing `linux` line, and pressed Ctrl+X to boot. `nomodeset` tells the kernel not to load the advanced display drivers and to use basic display mode. My first attempt typed it as a separate line, which dropped me into the GRUB shell. It has to be part of the `linux` line.

## Problem 3: No Wired Network at Install Time

**Symptom:** The installer's network screen showed only a placeholder address (`192.168.100.2`) and no DHCP lease.

**Diagnosis:** The Pavilion's only NIC is wired Ethernet, and there was no Ethernet in my dorm setup. Proxmox's `vmbr0` bridging also expects a wired NIC, so Wi-Fi isn't a viable long-term fix.

**Resolution:** I finished the install on the placeholder address, then bought a GL.iNet Beryl travel router to bridge dorm Wi-Fi to a wired port. That was already a planned purchase for network segmentation, so it solved both needs.

## Problem 4: Proxmox Unreachable After Connecting to the Router (and a Forgotten Password)

**Symptom:** The Pavilion and my laptop were both on the Beryl, but I couldn't reach the Proxmox web UI at `https://192.168.100.2:8006`. I also didn't remember the root password I set during install.

**Diagnosis:**
- The Beryl's client list showed my desktop (`alanpc`) at `192.168.8.163`, an address I had reserved for its MAC address earlier.
- Proxmox had never been told to use that address. It was still on the `192.168.100.2` placeholder from the install.
- The Beryl's LAN is `192.168.8.0/24`, so `192.168.100.2` was in a different subnet. Devices can only talk directly inside their own subnet and otherwise go through the gateway, and Proxmox couldn't even reach the gateway.

**Root cause:** A mismatch between a static placeholder IP and the real network. The DHCP reservation didn't help because a reservation only applies to devices that ask for an address over DHCP, and Proxmox was set to a fixed static address.

**Fix:** In `/etc/network/interfaces`, I changed the `vmbr0` block to `address 192.168.8.163/24` and `gateway 192.168.8.1`. I updated the matching line in `/etc/hosts` and set `nameserver 192.168.8.1` in `/etc/resolv.conf`. Then I ran `ifreload -a` to apply it without a reboot and `ping -c 3 192.168.8.1` to confirm the gateway was reachable. I kept it static, matching the reservation, because a hypervisor is infrastructure and its address shouldn't change.

**Lesson:** A static address and a DHCP reservation are two separate mechanisms. If they don't agree, the machine is unreachable even though everything looks configured.

### Problem 4b: Forgotten Root Password

**Symptom:** I couldn't log in to the host console or the web UI.

**Fix (password reset from the bootloader):**
1. At boot, in the GRUB menu, press `e` to edit the entry, and append ` init=/bin/bash` to the end of the `linux` line. Press Ctrl+X to boot.
2. The kernel starts a bash shell as root instead of running init, so no login is asked for.
3. The root filesystem comes up read-only, so `mount -o remount,rw /` makes it writable.
4. `passwd root` sets a new password.
5. `sync` flushes the change from memory to disk, because a forced reboot skips the clean shutdown.
6. `reboot -f` forces the reboot, because the normal `reboot` asks init to do it and init isn't running.

**Why it works:** The boot chain is firmware, then GRUB (the bootloader that loads the kernel), then the kernel, then init (process 1, which starts every other service and the login prompt). `init=/bin/bash` replaces that last step, so nothing checks who you are.

**Lesson:** Without disk encryption or a GRUB password, physical access to a machine is root access. This is fine for a home lab, but it's the reason a real server would use full-disk encryption and a locked bootloader.

## Problem 5: Web UI Unreachable Over Tailscale While SSH Worked

**Symptom:** From a different network, I could SSH into Proxmox over Tailscale, but the web UI wouldn't load.

**Diagnosis:** SSH working proved the Tailscale tunnel and host were fine, so the problem was in how I was reaching the web service. First I'd used the Beryl's LAN address (`192.168.8.163`), which only exists inside the Beryl's network and doesn't work from anywhere else. After switching to the Tailscale address (`100.x.x.x`), it still failed. I double-checked the IP against `tailscale ip -4` and it matched, which ruled out a typo.

**Root cause:** I left off the port. A service is an address plus a port. SSH defaults to port 22, so I never had to type it. A browser assumes port 443 for `https://`, but Proxmox's web UI listens on 8006.

**Fix:** `https://<tailscale-ip>:8006`. I clicked through the self-signed certificate warning, which is expected here since the traffic is already inside the encrypted tunnel.

**Lesson:** Troubleshooting splits into two questions: can the packet reach the service, and will the service let me in. A working SSH session ruled out the network, so the problem had to be the address and port on the web side.

## Problem 6: Locked Out of SSH After Disabling Password Authentication

**Symptom:** After setting `PasswordAuthentication no` and restarting sshd, a fresh `ssh root@<tailscale-ip>` returned `Permission denied (publickey)`.

**Diagnosis:** Server-side config showed the right values (`sshd -T` reported `passwordauthentication no` and `kbdinteractiveauthentication no`, so the PAM keyboard-interactive loophole was closed too). That meant the server was working as configured, but my key wasn't being accepted. I had been logging in with the password the whole time, so my public key had never been added to the host's `authorized_keys`.

**Root cause:** I disabled the fallback (password login) before proving the replacement (key login) worked in a fresh session. An earlier check of the config line had also been misleading: a `#` in front of a line makes it a comment, so the default (`yes`) applied, and `sshd -T` reads the config file rather than the running daemon.

**Fix:** My original session was closed, so I used the Proxmox web UI shell (node, then Shell) to append my public key to `~/.ssh/authorized_keys`, then ran `chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys`. The permissions matter because sshd refuses key files other users could write to, since anyone able to add a key could log in. Then I tested a fresh SSH login with the key.

**Verification:** A forced password attempt (`ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no root@<tailscale-ip>`) failed with `Permission denied (publickey)` without ever prompting for a password.

**Lesson:** Prove the replacement works, in a fresh session, before removing the fallback. I made the same mistake with the Raspberry Pi earlier, so it is worth making a habit.

## Result / Verification

- Proxmox VE 9.1 installed and running from the dedicated internal drive. Windows is untouched and dual-boot works.
- Web UI reachable at `https://<tailscale-ip>:8006` and SSH reachable over Tailscale from a different network than the Beryl.
- SSH is key-only, confirmed by the forced-password test above.
- Proxmox is the default UEFI boot entry, and key expiry is disabled for the node.

## What's Next

- Turn off the Beryl's Wi-Fi broadcast now that remote access works without it, keeping in mind the recovery path if the dorm captive portal drops the Beryl (physical Ethernet).
- Fix the apt repositories (disable the enterprise repo, add the no-subscription repo) and update.
- Create the first VM (Ubuntu Server 24.04 LTS) for the Wazuh manager, with plans for a second NIC on `vmbr1`.
- Add the 4TB external drive as Proxmox storage.
- Build Kali plus vulnerable targets on the isolated `vmbr1` bridge only.

---

*Lessons for next time: (1) when an installer fails identically on verified media, search the exact error text before re-diagnosing from scratch; (2) a static IP and a DHCP reservation are separate things, so make them agree; (3) before removing a fallback, prove the replacement works from a fresh session.*
