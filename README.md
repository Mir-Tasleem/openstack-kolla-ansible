# OpenStack 2025.2 Multinode Lab — Kolla-Ansible, Vagrant & VirtualBox

A reproducible record of deploying and validating an **OpenStack 2025.2** multinode environment on an Ubuntu laptop using **VirtualBox**, **Vagrant**, and **Kolla-Ansible**.

> **Scope and provenance:** The topology, configuration values, resource names, IP addresses, troubleshooting observations, and successful tests below are from the completed lab. Some setup and resource-creation commands are **reconstruction examples**, because the exact original commands or file contents were not retained in the conversation. Check the current project files before replaying those commands. Passwords, SSH private keys, and generated secrets are intentionally omitted.

## 1. Lab architecture

```text
Ubuntu laptop (VirtualBox + Vagrant + Ansible + OpenStack CLI)
│
├── Host-only management: vboxnet1 — 10.0.1.1/24
│   ├── controller01 eth1 — 10.0.1.10/24
│   ├── compute01    eth1 — 10.0.1.11/24
│   └── Kolla internal VIP — 10.0.1.100
│
├── Host-only external: vboxnet2 — 192.168.58.1/24
│   └── controller01 eth2 (UP, no IPv4 address)
│       └── OVS br-ex → Neutron router external gateway 192.168.58.122
│           └── Floating IP 192.168.58.110
│
└── VirtualBox NAT: eth0 — 10.0.2.15/24 (guest NAT interface)

controller01 (7 GiB RAM, 2 vCPU)      compute01 (5 GiB RAM, 2 vCPU)
  OpenStack APIs / Neutron L3           Nova compute / libvirt
  OVS br-ex, br-int, br-tun    <---->   OVS / VXLAN tenant networking
                                        └── vm01 (CirrOS)
                                            10.10.10.133/24
                                            floating: 192.168.58.110
```

### Host and VM details

| Item | Lab value |
|---|---|
| Host OS | Ubuntu 24.04.4 |
| Host resources | ~31 GiB RAM, 8 logical CPUs |
| VirtualBox | 7.1.18 |
| Vagrant | 2.4.9 |
| Guest OS | Ubuntu 24.04.3 |
| Controller VM | `kolla-controller01`, 7 GiB RAM, 2 vCPU, ~61 GB root disk |
| Compute VM | `kolla-compute01`, 5 GiB RAM, 2 vCPU, ~61 GB root disk |
| Kolla-Ansible | 21.3.1.dev5, stable/2025.2 |
| Ansible Core | 2.19.14 |
| Container engine | Docker |
| Neutron mechanism | Open vSwitch |
| Tenant overlay | VXLAN |
| Nova virtualization | KVM (`/dev/kvm` available on compute) |
| Working directory | `~/Desktop/openstack/kolla-lab` |
| Inventory | `ansible/kolla/multinode` |

The storage VM was excluded from this deployment, and **Cinder was disabled**.

## 2. Prerequisites

Install/configure the following on the Ubuntu host:

- VirtualBox and Vagrant.
- Python 3 virtual environment and pip.
- Ansible, Kolla-Ansible, and the OpenStack CLI in the project virtual environment.
- VirtualBox host-only adapters for the management and external networks.
- A Vagrant configuration defining the controller and compute nodes with the NICs and resources listed above.
- Nested virtualization / hardware acceleration so the compute VM exposes `/dev/kvm`.

> The original `Vagrantfile`, installation commands, and full inventory contents were not supplied in the retained conversation. Preserve the working copies in your repository; do not replace them with guessed templates.

### Start and inspect the VMs

From the lab directory, typical Vagrant operations are:

```bash
cd ~/Desktop/openstack/kolla-lab
vagrant status
vagrant up
vagrant ssh controller01  # use the exact VM name defined in your Vagrantfile
```

If the Vagrant names differ, use `vagrant status` to find the correct names. The VirtualBox VM names observed were `kolla-controller01` and `kolla-compute01`.

Verify the controller's interfaces and compute virtualization support:

```bash
ip -br addr
ls -l /dev/kvm
```

The compute node, not necessarily the controller, must expose `/dev/kvm` for KVM-backed Nova instances.

## 3. Prepare the deployment environment

Activate the lab's existing Python environment:

```bash
cd ~/Desktop/openstack/kolla-lab
source .venv/bin/activate
kolla-ansible --version
ansible --version
```

The working inventory was:

```text
ansible/kolla/multinode
```

SSH connectivity was configured using Vagrant's forwarded SSH ports. Verify access before deployment:

```bash
ansible -i ansible/kolla/multinode all -m ping
```

> The warning `Invalid characters were found in group names` appeared during ad hoc Ansible commands. It was non-blocking in this lab.

## 4. Configure Kolla-Ansible

The confirmed values in `/etc/kolla/globals.yml` were:

```yaml
kolla_base_distro: "ubuntu"
kolla_container_engine: "docker"

network_interface: "eth1"
api_interface: "eth1"
tunnel_interface: "eth1"
kolla_internal_vip_address: "10.0.1.100"

neutron_plugin_agent: "openvswitch"
neutron_external_interface: "eth2"

nova_compute_virt_type: "kvm"

enable_horizon: "yes"
enable_cinder: "no"
```

**Important network separation:**

- `eth0` provides VirtualBox NAT connectivity.
- `eth1` carries management/API/VXLAN traffic (`10.0.1.0/24`).
- `eth2` connects to the external provider network through `br-ex` and **must not have an IPv4 address**.
- The external IP `192.168.58.122` belongs to the Neutron router's `qg-*` interface, **not** to `eth2`.

Keep `/etc/kolla/passwords.yml` private; never commit it to Git.

## 5. Deploy OpenStack

The following Kolla-Ansible phases were completed successfully in the lab. These commands reproduce the phase order; ensure your inventory, passwords, and Kolla configuration are prepared first.

```bash
cd ~/Desktop/openstack/kolla-lab
source .venv/bin/activate

kolla-ansible -i ansible/kolla/multinode bootstrap-servers
kolla-ansible -i ansible/kolla/multinode prechecks
kolla-ansible -i ansible/kolla/multinode pull
kolla-ansible -i ansible/kolla/multinode deploy
kolla-ansible -i ansible/kolla/multinode post-deploy
```

After deployment, the controller had approximately **32 Docker containers** and the compute node **9**. Core services included Nova, Neutron, Glance, Keystone, Horizon, RabbitMQ, MariaDB, and Open vSwitch.

Load the generated OpenStack credentials **on the machine where the admin-openrc file is present**:

```bash
source /etc/kolla/admin-openrc.sh
```

Check API and service health:

```bash
openstack compute service list
openstack network agent list
openstack hypervisor list
openstack server list
```

The controller's Nova scheduler/conductor, compute service, and Neutron L3/DHCP/metadata/OVS agents were observed operational during the lab.

## 6. Create provider and private networks

The resulting resources were:

| Resource | Value |
|---|---|
| External network | `external-net` |
| External type | Flat provider network, physical network `physnet1` |
| External subnet | `external-subnet`: `192.168.58.0/24` |
| External gateway | `192.168.58.1` (Ubuntu host `vboxnet2`) |
| External allocation pool | `192.168.58.100`–`192.168.58.200` |
| External DHCP | Disabled |
| Private network | `private-net` |
| Private subnet | `private-subnet`: `10.10.10.0/24` |
| Private gateway | `10.10.10.1` |
| Router | `router1` |
| Router external gateway | `192.168.58.122` |
| Router SNAT | Enabled |

**Reconstruction example** for creating equivalent resources (not a verbatim transcript of the original commands):

```bash
openstack network create \
  --external --provider-network-type flat \
  --provider-physical-network physnet1 external-net

openstack subnet create \
  --network external-net \
  --subnet-range 192.168.58.0/24 \
  --gateway 192.168.58.1 \
  --no-dhcp \
  --allocation-pool start=192.168.58.100,end=192.168.58.200 \
  external-subnet

openstack network create private-net
openstack subnet create \
  --network private-net \
  --subnet-range 10.10.10.0/24 \
  --gateway 10.10.10.1 \
  private-subnet

openstack router create router1
openstack router add subnet router1 private-subnet
openstack router set router1 --external-gateway external-net
```

Neutron dynamically selected `192.168.58.122` for the router external interface in the original deployment; do not assume the same IP will be selected on a fresh deployment.

## 7. Upload an image and launch an instance

The lab successfully used:

| Item | Value |
|---|---|
| Image | `cirros-0.6.3` (ACTIVE) |
| Flavor | `m1.tiny` — 1 vCPU, 512 MiB RAM, 1 GiB disk |
| Keypair | `openstack-key` |
| VM | `vm01` |
| VM placement | `compute01` |
| VM private IP | `10.10.10.133` |
| Floating IP | `192.168.58.110` |

The default security group was configured to allow **ICMP** and **TCP port 22**.

**Reconstruction example:** after downloading the CirrOS 0.6.3 image and supplying its local path:

```bash
openstack image create cirros-0.6.3 \
  --file /path/to/cirros-0.6.3-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --public

openstack flavor create m1.tiny \
  --vcpus 1 --ram 512 --disk 1

openstack keypair create --public-key ~/.ssh/openstack_lab.pub openstack-key

openstack security group rule create --protocol icmp default
openstack security group rule create --protocol tcp --dst-port 22 default

openstack server create \
  --image cirros-0.6.3 \
  --flavor m1.tiny \
  --network private-net \
  --key-name openstack-key \
  vm01

openstack floating ip create external-net
openstack server add floating ip vm01 <allocated-floating-ip>
```

**Notes:** The image file path above is a placeholder. Security group rules might already exist; avoid duplicating them. The public key path must match your actual keypair. On a fresh deployment, floating and private IPs can differ from the values shown in this README.

Verify:

```bash
openstack server show vm01
openstack floating ip list
openstack server list
```

## 8. Validate connectivity

### From the Ubuntu laptop

The host routes external traffic through `vboxnet2` (`192.168.58.1`):

```bash
ip route get 192.168.58.110
ping -c 4 192.168.58.110
ip neigh show 192.168.58.110
ssh -i ~/.ssh/openstack_lab -o StrictHostKeyChecking=accept-new cirros@192.168.58.110
```

**Observed success:** 4/4 ICMP replies, **0% packet loss**. SSH login to CirrOS succeeded.

The ARP neighbor entry was:

```text
192.168.58.110 dev vboxnet2 lladdr fa:16:3e:dc:f4:3e STALE
```

`STALE` is a normal ARP neighbor-cache state, not a failure. The MAC belongs to the Neutron router's external interface.

### Inside CirrOS

```bash
ip addr
ip route
ping -c 4 10.10.10.1
```

Observed VM interface:

```text
eth0: UP, MTU 1450
inet 10.10.10.133/24
```

Observed routes:

```text
default via 10.10.10.1 dev eth0 src 10.10.10.133
10.10.10.0/24 dev eth0 scope link src 10.10.10.133
169.254.169.254 via 10.10.10.2 dev eth0 src 10.10.10.133
```

The `169.254.169.254` route provides access to OpenStack metadata. The guest MTU of **1450** is consistent with VXLAN encapsulation overhead. The gateway ping succeeded **4/4**.

## 9. Troubleshooting: floating IP unreachable

### Symptom

Initially, the Ubuntu host could not ping `192.168.58.110`. The host ARP neighbor state was `FAILED`. The Neutron router could not ping host gateway `192.168.58.1` either.

### Investigation

Inspect the router namespace and interfaces on the controller:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m shell \
  -a "ip netns list"

ansible controller01 -i ansible/kolla/multinode -b -m shell \
  -a "ip netns exec qrouter-5f14ff35-8088-4271-8ef5-7861f093063b ip -br addr"
```

The router's external interface was `qg-49f32f08-e9` with `192.168.58.122/24` and `192.168.58.110/32`.

The router **did generate ARP requests**, confirmed by a capture inside its namespace:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m shell \
  -a "timeout 15 ip netns exec qrouter-5f14ff35-8088-4271-8ef5-7861f093063b tcpdump -eni qg-49f32f08-e9 arp"
```

Captured packets included:

```text
fa:16:3e:dc:f4:3e > ff:ff:ff:ff:ff:ff, ARP:
Request who-has 192.168.58.1 tell 192.168.58.122
```

Open vSwitch also showed `eth2` attached to `br-ex` and patch connectivity between `br-ex` and `br-int`:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "docker exec openvswitch_vswitchd ovs-vsctl show"
```

However, an attempted packet capture on the controller's physical external interface returned:

```text
tcpdump: eth2: That device is not up
```

**Root cause:** `controller01`'s `eth2` was **DOWN** following a VirtualBox VM restart. OVS bridge membership alone did not ensure the underlying NIC was operational.

### Immediate fix

```bash
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "ip link set eth2 up"
```

Connectivity to the floating IP immediately recovered.

### VirtualBox promiscuous mode

The controller's third NIC is attached to the host-only external network (`vboxnet2`). Its promiscuous policy was set to **Allow All** to permit Neutron/OVS traffic using guest MAC addresses:

```bash
# Run with the VM powered off; then start it again.
VBoxManage modifyvm kolla-controller01 --nicpromisc3 allow-all
```

The observed VirtualBox NIC 3 MAC was `08:00:27:31:61:e6`. The external router MAC was `fa:16:3e:dc:f4:3e`.

### Persistent fix: Netplan

The controller initially had these Netplan files:

```text
/etc/netplan/01-netcfg.yaml      # eth0: DHCP
/etc/netplan/50-cloud-init.yaml  # enp0s3: DHCP (existing guest config)
/etc/netplan/50-vagrant.yaml     # eth1: 10.0.1.10/24
```

A separate configuration was added at `/etc/netplan/60-kolla-external.yaml`:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth2:
      dhcp4: false
      dhcp6: false
      optional: true
```

Create it with restricted permissions:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m shell \
  -a "printf 'network:\n  version: 2\n  renderer: networkd\n  ethernets:\n    eth2:\n      dhcp4: false\n      dhcp6: false\n      optional: true\n' > /etc/netplan/60-kolla-external.yaml && chmod 600 /etc/netplan/60-kolla-external.yaml"
```

`netplan generate` initially warned that `/etc/netplan/50-vagrant.yaml` had overly permissive mode (`0644`). Fix that and validate:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "chmod 600 /etc/netplan/50-vagrant.yaml"

ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "netplan generate"

# Applying network changes remotely can interrupt connectivity.
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "netplan apply"
```

Verify `eth2` is UP and has no IPv4 address:

```bash
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "ip -br addr show eth2"
```

Observed:

```text
eth2  UP  fe80::a00:27ff:fe31:61e6/64
```

After applying the configuration, `ping -c 4 192.168.58.110` again succeeded with **0% packet loss**. The later tests also showed `eth2` UP and the floating IP reachable. **The provided output does not independently include the controller reboot command**, so persistence across a completed reboot should be verified explicitly if not already done.

### Optional reboot validation

```bash
ansible controller01 -i ansible/kolla/multinode -b -m reboot \
  -a "reboot_timeout=300"

ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "ip -br addr show eth2"

ping -c 4 192.168.58.110
```

Wait for OpenStack containers to recover before treating a transient post-reboot ping failure as a persistent networking problem.

## 10. Operational verification checklist

```bash
# On Ubuntu host, with venv activated and OpenStack credentials loaded
openstack compute service list
openstack network agent list
openstack server list
openstack floating ip list

# Check external NIC on controller
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "ip -br addr show eth2"

# Check OVS bridges on controller
ansible controller01 -i ansible/kolla/multinode -b -m command \
  -a "docker exec openvswitch_vswitchd ovs-vsctl show"

# External instance reachability
ping -c 4 192.168.58.110
ssh -i ~/.ssh/openstack_lab cirros@192.168.58.110
```

## 11. Key lessons learned

1. **A Neutron resource can be ACTIVE while actual packet forwarding is broken.** Test ARP, ICMP, and SSH, not just API status.
2. **OVS bridge membership does not guarantee the physical NIC is UP.** Verify `ip -br link show eth2`.
3. **Do not assign a host IPv4 address to `eth2`** when Kolla/Neutron uses it as the external provider bridge port.
4. **VirtualBox promiscuous mode matters** when the VM forwards traffic with MAC addresses other than its virtual NIC MAC.
5. **Use packet captures at successive layers** (router `qg-*`, controller `eth2`, host `vboxnet2`) to isolate where traffic stops.
6. **Persist interface state with Netplan** rather than relying on a manual `ip link set ... up` after each reboot.
7. **Separate tenant addressing from floating IPs.** CirrOS owns `10.10.10.133`; Neutron translates the external `192.168.58.110` address.
8. **Keep deployment secrets out of version control.** Avoid committing `/etc/kolla/passwords.yml`, `admin-openrc.sh`, or private SSH keys.

## 12. Suggested repository structure

```text
kolla-lab/
├── README.md
├── Vagrantfile                 # actual working VM definitions
├── ansible/
│   └── kolla/
│       └── multinode          # working Ansible inventory
├── config/
│   ├── globals.yml.example    # sanitized Kolla settings
│   └── 60-kolla-external.yaml # persistent external NIC setup
├── scripts/                   # optional repeatable setup/check scripts
└── .gitignore
```

Recommended `.gitignore` entries:

```gitignore
.venv/
.vagrant/
*.log
*passwords.yml
*admin-openrc.sh
*.pem
*.key
```

---

**Final confirmed result:** `vm01` was ACTIVE on `compute01`, accessible from the Ubuntu laptop through floating IP `192.168.58.110`, and reachable over SSH. Its private address was `10.10.10.133`, its default gateway `10.10.10.1`, and its external connectivity was restored by activating and configuring controller `eth2`.
