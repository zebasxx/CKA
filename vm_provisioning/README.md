# CKA training VM provisioning

This directory provisions a compact libvirt lab for the
[CKA training calendar](<../Training Docs/Training Calendar.md>). It creates a
reusable Ubuntu template and four VMs:

| VM | Purpose | vCPU | RAM | Disk | Address | Normal state |
| --- | --- | ---: | ---: | ---: | --- | --- |
| `cka-cp01` | Primary cluster control plane | 2 | 3 GiB | 30 GiB | `192.168.124.51` | Running |
| `cka-worker01` | Primary cluster worker | 2 | 3 GiB | 30 GiB | `192.168.124.61` | Running |
| `cka-worker02` | Primary cluster worker | 2 | 3 GiB | 30 GiB | `192.168.124.62` | Running |
| `cka-context01` | Secondary single-node cluster | 2 | 3 GiB | 30 GiB | `192.168.124.71` | Powered off |

The primary cluster uses **6 vCPUs, 9 GiB RAM, and three virtual disks** while
running. The secondary cluster raises the total to **8 vCPUs and 12 GiB RAM**
only when it is started for multi-context practice. The powered-off template
does not consume CPU or RAM.

The four sparse clone disks plus the template have 150 GiB of virtual capacity,
but qcow2 files consume physical storage only as data is written.

## Why four VMs

One control plane and two workers are enough for the course exercises involving:

- `kubeadm` installation, joining, reset, and upgrades;
- draining a worker while workloads move to another worker;
- labels, taints, affinity, topology, and scheduling failures;
- CNI networking, NetworkPolicies, Services, DNS, Ingress, and Gateway API;
- local persistent storage and CSI exercises;
- kubelet, worker, and control-plane failure recovery.

The fourth VM exists only for the multi-context drill. Configure it as a
single-node kubeadm cluster and remove its control-plane scheduling taint when
workloads need to run there. Keep it powered off on all other training days.

This course does not require:

- a highly available three-control-plane cluster;
- external control-plane load balancers;
- dedicated ingress workers;
- GitLab;
- TrueNAS or a separate storage appliance.

Static `hostPath` exercises and a lightweight local-path CSI provisioner are
sufficient for the storage module. High availability and external storage are
valuable production topics but would add resource usage without being required
by this training path.

## What the playbooks do

- `create_template_playbook.yml` installs Ubuntu once, generalizes it, and
  leaves the golden template powered off.
- `clone_vms_playbook.yml` makes full, independent clones and writes their
  hostname and network configuration directly to their offline disks.
- `tasks/network.yml` creates the dedicated `k8s-lab` NAT network and fixed DHCP
  reservations.

The playbooks provision operating systems only. They do not install a container
runtime, Kubernetes, a CNI, an ingress or Gateway controller, a metrics server,
or a storage provisioner. Installing and troubleshooting those components is
part of the training.

Clones are not configured for host-boot autostart. The three primary-cluster
VMs start after initial cloning; `cka-context01` remains powered off.

## Host requirements

Recommended capacity for running the primary cluster alongside the desktop and
libvirt host:

- Hardware virtualization enabled in firmware.
- 8 or more logical CPUs.
- 16 GiB or more host RAM; 24 GiB provides more headroom.
- Approximately 80 GiB of free physical disk for the initially sparse images,
  snapshots, ISO, container images, and logs.
- A Linux host with KVM/libvirt.

The current host has 22 logical CPUs, 61 GiB RAM, and sufficient free storage,
so this lab size leaves comfortable headroom.

## Prerequisites

Install the host packages used by the playbooks:

```bash
sudo apt update
sudo apt install ansible-core libguestfs-tools libvirt-clients \
  libvirt-daemon-system python3-libvirt python3-lxml qemu-utils virt-manager \
  virtinst xorriso
sudo usermod -aG libvirt "$USER"
```

Log out and back in after adding the user to the `libvirt` group.

Install the Ansible collection:

```bash
cd /home/seb/Code/4-GitHub-Seba/CKA/vm_provisioning
ansible-galaxy collection install -r requirements.yml
```

The guests use the local Ed25519 public key configured by
`ssh_public_key_file` in `vars.yml`. Create it only if it does not exist:

```bash
test -f "$HOME/.ssh/id_ed25519.pub" || \
  ssh-keygen -t ed25519 -f "$HOME/.ssh/id_ed25519"
ssh-keygen -lf "$HOME/.ssh/id_ed25519.pub"
```

Do not replace an existing key; it may already identify the user to other
systems.

Review the paths and settings in `vars.yml`:

```bash
grep -E '^(vm_directory|ubuntu_iso|vm_memory_mb|vm_disk_gb):' vars.yml
test -f /var/vms/ISOs/ubuntu-26.04.1-live-server-amd64.iso
```

The VM directory can be any local directory. Ensure that the configured Ubuntu
Server ISO exists and has been verified against its publisher's checksum.

## Important: existing copied-lab domains

The provisioning workflow is deliberately non-destructive. It skips a domain
when a VM with the same name already exists and does not resize or repurpose it.
Changing `vars.yml` therefore does **not** shrink existing VMs.

Before creating the compact lab, inspect existing domains:

```bash
virsh -c qemu:///system list --all
virsh -c qemu:///system dominfo k8s-ubuntu-template
```

The new `cka-*` names, MAC addresses, and fixed IP addresses do not overlap the
previous `k8s-*` guests. The labs can therefore coexist on the existing
`k8s-lab` libvirt network while sharing the powered-off
`k8s-ubuntu-template`. The previous lab does not need to be deleted. Keep its
VMs powered off when they are not in use so they consume no CPU or RAM.

Old VMs that are no longer needed can at least be shut down to release RAM:

```bash
virsh -c qemu:///system list --name
virsh -c qemu:///system shutdown <old-vm-name>
```

Never delete a domain or disk solely because its name is absent from the new
four-VM definition.

## Build the template and VMs

Check syntax first:

```bash
cd /home/seb/Code/4-GitHub-Seba/CKA/vm_provisioning
ansible-playbook create_template_playbook.yml --syntax-check
ansible-playbook clone_vms_playbook.yml --syntax-check
```

Build the reusable template and clone the lab:

```bash
ansible-playbook create_template_playbook.yml --ask-become-pass \
  --extra-vars ansible_become_exe=/usr/bin/sudo.ws
ansible-playbook clone_vms_playbook.yml --ask-become-pass \
  --extra-vars ansible_become_exe=/usr/bin/sudo.ws
```

The `sudo.ws` override handles the compatibility `sudo` executable on the
current host. Omit the extra variable on systems where normal `sudo` works with
Ansible.

Messages saying no inventory was parsed are expected: both playbooks run
against Ansible's implicit localhost.

The clone playbook:

- requires the template's `.ready` marker;
- refuses to overwrite orphaned disk files;
- leaves existing domains unchanged;
- creates full clones that do not depend on the template disk;
- starts the three primary-cluster nodes;
- leaves the secondary practice cluster powered off.

Inspect the result:

```bash
virsh -c qemu:///system list --all
virsh -c qemu:///system net-dhcp-leases k8s-lab
ssh ubuntu@192.168.124.51
```

## Suggested cluster layout

Build the main cluster during Days 01–05:

```text
cka-cp01       control plane    192.168.124.51
cka-worker01   worker           192.168.124.61
cka-worker02   worker           192.168.124.62
```

Use a Pod CIDR that does not overlap `192.168.124.0/24`. Select a CNI that
enforces NetworkPolicies because those policies are part of the networking and
simulation modules.

Install lightweight add-ons only when their exercises require them:

- a metrics server for HPA and `kubectl top`;
- an Ingress controller;
- a Gateway API implementation;
- a local-path dynamic storage provisioner.

For Days 31–35, start the fourth VM and build a separate single-node cluster:

```bash
virsh -c qemu:///system start cka-context01
ssh ubuntu@192.168.124.71
```

After `kubeadm init`, allow ordinary workloads on that single node if required:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Copy its kubeconfig under a distinct context name and merge it with the primary
cluster kubeconfig for the multi-context drill. Shut it down after practice:

```bash
virsh -c qemu:///system shutdown cka-context01
```

## Operating the lab

Start the primary cluster:

```bash
for vm in cka-cp01 cka-worker01 cka-worker02; do
  virsh -c qemu:///system start "$vm"
done
```

Shut it down gracefully, workers first:

```bash
for vm in cka-worker01 cka-worker02 cka-cp01; do
  virsh -c qemu:///system shutdown "$vm"
done
```

Confirm that the guests have stopped before shutting down the host:

```bash
virsh -c qemu:///system list --all
```

## Snapshots

Create clean snapshots at useful milestones, such as after the operating system
clone, after Kubernetes installation, and before destructive troubleshooting
drills. Shut down the target VM before taking an external training snapshot:

```bash
virsh -c qemu:///system shutdown <vm-name>
virsh -c qemu:///system snapshot-create-as \
  --domain <vm-name> \
  --name <snapshot-name> \
  --description "CKA training checkpoint" \
  --atomic
```

List snapshots before restoring or deleting any of them:

```bash
virsh -c qemu:///system snapshot-list <vm-name>
```

Treat snapshot restoration as destructive to changes made after the snapshot.
Keep important manifests and notes in this repository rather than only inside
the guests.
