# Ansible Role: Ludus Dual Home

An Ansible role that adds a second NIC to [Ludus](https://ludus.cloud/) VMs, making them dual-homed across VLANs.

## The Problem

Ludus VMs get **one NIC on one VLAN**. That's great until you need a VM that can talk to two different networks -- like a jump box that sits between an attacker network and a target network. Normally you'd have to manually add NICs through Proxmox and configure interfaces by hand on every deploy.

## What This Role Does

Drop it into your range config and it handles everything automatically:

1. Adds a second network interface to the VM via the Proxmox API
2. Assigns it to whatever VLAN you specify
3. Configures a static IP and brings the interface up

That's it. One role, one variable (`ludus_dual_home_vlan`), and your VM is on two networks.

## Use Cases

**Pivot labs** -- Build a range where attackers must pivot through jump boxes to reach targets on a separate VLAN. The attack box can't reach the targets directly; it has to go through the dual-homed machines first. See [PivotLab](https://github.com/BLTSEC/PivotLab) for a full working example that uses this role with 11 pivoting tools.

```
Attacker (VLAN 20) --> Jump Box (VLAN 20 + VLAN 22) --> Targets (VLAN 22)
```

**Simulating real enterprise networks** -- Corporate environments almost always have servers that span multiple network segments (DMZs, management VLANs, database VLANs). Dual-homing lets you replicate that topology in Ludus.

**Firewall/IDS testing** -- Put a monitoring VM on two VLANs to capture and analyze traffic flowing between network segments.

**Active Directory labs** -- Place domain controllers or management servers on multiple VLANs so different subnets can authenticate without routing everything through the Ludus router.

## Quick Start

```sh
ludus ansible role add BLTSEC.ludus_dual_home
```

Then add it to any VM in your range config:

```yaml
- vm_name: "{{ range_id }}-jumpbox"
  hostname: "jumpbox"
  template: debian-12-x64-server-template
  vlan: 20
  ip_last_octet: 220
  linux: true
  roles:
    - BLTSEC.ludus_dual_home
  role_vars:
    ludus_dual_home_vlan: 22
```

Deploy and the VM gets two interfaces: `10.X.20.220` on VLAN 20 and `10.X.22.220` on VLAN 22.

## Requirements

- Ludus range server (v1.5+)
- `community.general` Ansible collection (for `proxmox_nic` module)
- `jq` installed on the Ludus host
- Target VMs must be Linux (Debian/Ubuntu)

> **Note:** This role relies on variables automatically provided by the Ludus framework at runtime: `proxmox_vmid`, `range_second_octet`, `ludus` (the range config list), and `hostvars['localhost']` Proxmox API credentials. These do not need to be set manually -- Ludus injects them into the playbook context.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

    # The VLAN number to add as the second interface
    ludus_dual_home_vlan: 22

    # The last octet for the IP on the second VLAN (defaults to same as primary interface)
    ludus_dual_home_ip_last_octet: "{{ (ludus | selectattr('vm_name', 'match', inventory_hostname) | first).ip_last_octet }}"

The IP address is automatically constructed as `10.<range_second_octet>.<ludus_dual_home_vlan>.<ludus_dual_home_ip_last_octet>/24`.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: jump_boxes
  roles:
    - BLTSEC.ludus_dual_home
  vars:
    ludus_dual_home_vlan: 22
```

## Example Ludus Range Config

```yaml
ludus:
  - vm_name: "{{ range_id }}-jumpbox"
    hostname: "{{ range_id }}-jumpbox"
    template: debian-12-x64-server-template
    vlan: 20
    ip_last_octet: 220
    ram_gb: 4
    cpus: 2
    linux: true
    roles:
      - BLTSEC.ludus_dual_home
    role_vars:
      ludus_dual_home_vlan: 22
```

This will give the VM two interfaces:
- **Primary**: `10.<range>.20.220/24` (VLAN 20)
- **Secondary**: `10.<range>.22.220/24` (VLAN 22)

## Ludus Setup

```sh
# Install the role
ludus ansible role add BLTSEC.ludus_dual_home

# Edit your range config to add the role (see example above)
ludus range config get > config.yml
# ... edit config.yml ...
ludus range config set -f config.yml

# Deploy (only user-defined roles)
ludus range deploy -t user-defined-roles
```

## How It Works

1. **Checks** if `net1` already exists on the VM (idempotent)
2. **Adds** a second virtio NIC via the Proxmox API, tagged to the specified VLAN
3. **Waits** for the hotplugged NIC to be detected by the guest OS
4. **Discovers** the new interface name by matching the MAC address
5. **Configures** a static IP in `/etc/network/interfaces`
6. **Brings up** the interface

## Security Considerations

The second NIC communicates **directly at Layer 2** on the target VLAN. Traffic on this interface does **not** traverse the Ludus router VM, which means:

- **Ludus firewall rules** (inter-VLAN rules, testing mode restrictions) do not apply to traffic on the second interface.
- If IP forwarding is enabled on the dual-homed VM (`sysctl net.ipv4.ip_forward=1`), it can act as a router between the two VLANs, allowing other VMs on the primary VLAN to reach the second VLAN without going through the Ludus router.

This is the expected and intended behavior for a dual-homed pivot lab. However, if you are using this role outside of a pivot/pentest lab context, be aware that the dual-homed VM effectively bridges two network segments.

### API Access

This role uses the same Proxmox API access patterns that Ludus core uses internally to add NICs to its own router VM. All API calls are **local only** (`https://127.0.0.1:8006`) -- the role talks to the Proxmox instance running on the Ludus host itself over loopback. No outbound internet calls are made.

Specifically, the role uses `delegate_to: localhost` to run commands on the Ludus host (not inside guest VMs). It authenticates with a time-limited `PVEAuthCookie` session ticket and Proxmox API credentials, both provided by the Ludus framework at runtime via `hostvars['localhost']`. The self-signed TLS certificate on localhost is expected and handled with `-k` (same as Ludus core). All tasks that handle API cookies or credentials have `no_log: true` set to prevent them from appearing in Ansible output. No credentials are transmitted to guest VMs.

## License

GPLv3

## Author Information

This role was created by [BLTSEC](https://github.com/BLTSEC), for use with [Ludus](https://ludus.cloud/).
