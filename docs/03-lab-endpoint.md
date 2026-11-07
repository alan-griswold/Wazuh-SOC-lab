# Building a Monitored Endpoint on the Isolated Lab Network

**One-line summary:** Built a reusable Ubuntu cloud-init template on Proxmox, cloned it onto the isolated lab bridge as the first monitored endpoint, got a Wazuh agent onto a machine with no internet access, and generated my first real detections.

## Goal

Give the Wazuh manager something to monitor. The endpoint has to live on `vmbr1`, the isolated bridge with no uplink, so that anything I later run against it (attacks, vulnerable software) can't reach the rest of my network. I also wanted a way to create new lab VMs in seconds instead of running an OS installer each time, and to do as much of it as possible from the command line.

## Architecture

- **`ubuntu-template` (VM 9000):** an Ubuntu 24.04 cloud image imported as a disk, 2 vCPUs, 2 GiB RAM, 20 GiB disk, one NIC on `vmbr1`, plus a cloud-init drive. Converted to a Proxmox template, so it can be cloned but not started.
- **`lab-ubuntu-01` (VM 101):** a full clone of the template with the static address `10.10.10.30/24` and no gateway. It sits only on `vmbr1`.
- **Wazuh agent 4.14.9** on the endpoint, enrolled with the manager at `10.10.10.10`. The agent version matches the manager's, since an agent can't be newer than its manager.

Isolation check: the endpoint can reach the Wazuh VM (`10.10.10.10`) but not the internet. The Wazuh VM, which has a NIC on both bridges, doubles as a temporary file server so software can be moved in without giving the endpoint a route out.

## Steps Taken

1. Downloaded the Ubuntu cloud image on the Proxmox host and verified it with `sha256sum -c SHA256SUMS --ignore-missing`.
2. Created the template shell with `qm create`, imported the image with `qm disk import`, attached it as `scsi0`, and resized the disk to 20G.
3. Added a cloud-init drive (`--ide2 local-lvm:cloudinit`) and set the user and password (the password was set in the web UI so it stayed out of shell history).
4. Converted it with `qm template 9000`.
5. Cloned it with `qm clone 9000 101 --name lab-ubuntu-01 --full`, set the address with `qm set 101 --ipconfig0 ip=10.10.10.30/24`, and started it.
6. Verified inside the VM: the address, a successful ping to the Wazuh VM, a failed ping to `8.8.8.8` (the isolation working), and about 19G of usable disk (cloud-init grew the filesystem into the added space).
7. Got the agent package onto the endpoint (see the challenge below) and installed it.
8. Confirmed the agent showed as **Active** in the dashboard and generated test events.

## Challenge: Installing Software on a Machine With No Internet

**Problem:** The Wazuh agent installs from `packages.wazuh.com`, and the endpoint, by design, can't reach the internet.

**Options considered:** temporarily attach a second NIC on `vmbr0` (simple, but breaks the isolation, even briefly); set up a package proxy (right at scale, overkill for one VM); or stage the file on the one machine that can reach both sides.

**Solution:** The Wazuh VM already sits on both bridges, so I downloaded the package there and served it to the endpoint over `vmbr1` for a minute:

```
# on the Wazuh VM
curl -fsSLO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.9-1_amd64.deb
python3 -m http.server 8000 --bind 10.10.10.10

# on the endpoint
curl -fsSLO http://10.10.10.10:8000/wazuh-agent_4.14.9-1_amd64.deb
sudo WAZUH_MANAGER='10.10.10.10' dpkg -i ./wazuh-agent_4.14.9-1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```
`python3 -m http.server` is Python's built-in file server. Binding it to `10.10.10.10` makes it listen only on the lab interface, so it isn't exposed on the management network, and I stopped it with `Ctrl+C` once the file was copied.

## Smaller Mistakes

- **Created the template with no options.** I ran `qm create 9000` with nothing after it. `qm set 9000 --name ... --memory ...` adds or changes options on an existing VM, so there was no need to start over.
- **`curl: (35) SSL routines::wrong version number`.** I used `https://` against the Python file server, which speaks plain HTTP. When a TLS client reports "wrong version number", the first thing to check is an HTTP/HTTPS mismatch.

## Result / Verification

- The agent shows **Active** in the Wazuh dashboard under Agents, reporting from `10.10.10.30`.
- A burst of failed SSH logins for a non-existent user, sent from the Wazuh VM:
  ```
  for i in $(seq 1 15); do ssh -o BatchMode=yes -o StrictHostKeyChecking=no baduser@10.10.10.30 true; done
  ```
  produced many low-severity alerts, and after enough of them in a short window, higher-severity ones. Individual failures are low severity because on their own they're noise, and the pattern is what escalates.

## What's Next

- Read the brute-force rule in `/var/ossec/ruleset/rules/` and write a custom rule of my own in `/var/ossec/etc/rules/local_rules.xml`. Rules in `ruleset/` get overwritten on updates, so custom ones go in `etc/rules/`.
- Add a Windows endpoint and a Kali VM on `vmbr1`.
- Decide how the Raspberry Pi honeypot reports in without sitting on the same LAN as my real devices.
- Reserve the Wazuh VM's `ens18` address on the Beryl for its MAC, since it's DHCP and could change.
- Change the generated Wazuh admin password.

---

*Lessons for next time: (1) a template plus cloud-init makes new lab VMs a one-minute job; (2) controlled one-way staging beats opening a route into an isolated network; (3) read what you're actually matching before a command fails silently, since `grep` plus `>` makes an empty file when nothing matches; (4) single events are noise, and correlation rules are where detection happens.*
