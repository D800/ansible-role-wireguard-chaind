<!--
Copyright (C) 2018-2024 Robert Wimmer
Copyright (C) 2019 fbourqui
SPDX-License-Identifier: GPL-3.0-or-later
-->

# ansible-role-wireguard

This Ansible role is used in my blog series [Kubernetes the not so hard way with Ansible](https://www.tauceti.blog/post/kubernetes-the-not-so-hard-way-with-ansible-wireguard/) but can be used standalone of course. I use WireGuard and this Ansible role to setup a fully meshed VPN between all nodes of my little Kubernetes cluster.

In general WireGuard is a network tunnel (VPN) for IPv4 and IPv6 that uses UDP. If you need more information about [WireGuard](https://www.wireguard.io/) you can find a good introduction here: [Installing WireGuard, the Modern VPN](https://research.kudelskisecurity.com/2017/06/07/installing-wireguard-the-modern-vpn/).

## Supported operating systems

### Linux

This role should work with:

- Ubuntu 20.04 (Focal Fossa)
- Ubuntu 22.04 (Jammy Jellyfish)
- Ubuntu 24.04 (Noble Numbat)
- Archlinux
- Debian 11 (Bullseye)
- Debian 12 (Bookworm)
- Fedora 39
- AlmaLinux 9
- Rocky Linux 9
- openSUSE Leap 15.5
- openSUSE Leap 15.6
- Oracle Linux 9

### Linux - Best effort

- AlmaLinux 8
- Rocky Linux 8
- elementary OS 6
- CentOS 7 (end of life since end June 2024)

### MacOS

While this playbook configures, enables and starts a `systemd` service on Linux in a such a way that no additional action is needed, on MacOS it installs the required packages and it just generates the correct `wg0.conf` file that is then placed in the specified `wireguard_remote_directory` (`/opt/local/etc/wireguard` by default). In order to run the VPN, then, you need to:

```bash
sudo wg-quick up wg0
```

and to deactivate it

```bash
sudo wg-quick down wg0
```

or you can install the [official app](https://apps.apple.com/it/app/wireguard/id1451685025?l=en&mt=12) and import the `wg0.conf` file.

## Versions

I tag every release and try to stay with [semantic versioning](http://semver.org). If you want to use the role I recommend to checkout the latest tag. The master branch is basically development while the tags mark stable releases. But in general I try to keep master in good shape too.

## Requirements

By default port `51820` (protocol UDP) should be accessible from the outside. But you can adjust the port by changing the variable `wireguard_port`. Also IP forwarding needs to be enabled e.g. via `echo 1 > /proc/sys/net/ipv4/ip_forward`. I decided not to implement this task in this Ansible role. IMHO that should be handled elsewhere.
You can use my [ansible-role-harden-linux](https://github.com/githubixx/ansible-role-harden-linux) e.g. Besides changing sysctl entries (which you need to enable IP forwarding) it also manages firewall settings among other things.
Nevertheless the `PreUp`, `PreDown`, `PostUp` and `PostDown` hooks may be a good place to do some network related stuff before a WireGuard interface comes up or goes down.

## Changelog

**Change history:**

See full [CHANGELOG.md](https://github.com/githubixx/ansible-role-wireguard/blob/master/CHANGELOG.md)

**Recent changes:**

## 17.1.0

- **FIXES**
  - add missing `wg-config` tag ([Issue #211](https://github.com/githubixx/ansible-role-wireguard/issues/211))
  - hide peers with empty endpoints ([Issue #101](https://github.com/githubixx/ansible-role-wireguard/issues/101) - contribution by @Miroka96)

- **FEATURE**
  - add support for `Fedora 40`
  - add [Netplan](https://netplan.io/) support for Ubuntu (contribution by @kbcz1989)

- **OTHER**
  - update `.yamllint`
  - fix `ansible-lint` issues
  - update `.gitignore`

## 17.0.0

- **BREAKING**
  - removed support for `openSUSE 15.4` (reached end of life)

- **FEATURE**
  - add support for `Ubuntu 24.04`
  - add support for `openSUSE 15.6`

- **MOLECULE**
  - remove outdated `Proxmox` code
  - replace Vagrant box `rockylinux/9` with `bento/rockylinux-9`
  - use `ansible.builtin.package` for AlmaLinux
  - remove `AlmaLinux 8`, `Rocky Linux 8` and `CentOS 7` (outdated Python makes it hard to test with Ansible)

## Installation

- Directly download from Github (change into Ansible role directory before cloning):
`git clone https://github.com/githubixx/ansible-role-wireguard.git githubixx.ansible_role_wireguard`

- Via `ansible-galaxy` command and download directly from Ansible Galaxy:
`ansible-galaxy role install githubixx.ansible_role_wireguard`

- Create a `requirements.yml` file with the following content (this will download the role from Github) and install with
`ansible-galaxy role install -r requirements.yml`:

```yaml
---
roles:
  - name: githubixx.ansible_role_wireguard
    src: https://github.com/githubixx/ansible-role-wireguard.git
    version: 17.0.0
```

## Role Variables

These variables can be changed in `group_vars/` e.g.:

```yaml
# Directory to store WireGuard configuration on the remote hosts
wireguard_remote_directory: >-
  {%- if wireguard_ubuntu_use_netplan -%}
  /etc/netplan
  {%- elif ansible_os_family == 'Darwin' -%}
  /opt/local/etc/wireguard
  {%- else -%}
  /etc/wireguard
  {%- endif %}

# The default port WireGuard will listen if not specified otherwise.
wireguard_port: "51820"

# The default interface name that WireGuard should use if not specified otherwise.
wireguard_interface: "wg0"

# The default owner of the wg.conf file
wireguard_conf_owner: root

# The default group of the wg.conf file
wireguard_conf_group: "{{ 'root' if not ansible_os_family == 'Darwin' else 'wheel' }}"

# The default mode of the wg.conf file
wireguard_conf_mode: 0600

# Whether any change to the wg.conf file should be backup
wireguard_conf_backup: false

# The default state of the wireguard service
wireguard_service_enabled: "yes"
wireguard_service_state: "started"

# By default "wg syncconf" is used to apply WireGuard interface settings if
# they've changed. Older WireGuard tools doesn't provide this option. In that
# case as a fallback the WireGuard interface will be restarted. This causes a
# short interruption of network connections.
#
# So even if "false" is the default, the role figures out if the "syncconf"
# option of the "wg" utility is available and if not falls back to "true"
# (which means interface will be restarted as this is the only possible option
# in this case).
#
# Possible options:
# - false (default)
# - true
#
# Both options have their pros and cons. The default "false" option (do not
# restart interface)
# - does not need to restart the WireGuard interface to apply changes
# - does not cause a short VPN connection interruption when changes are applied
# - might cause network routes are not properly reloaded
#
# Setting the option value to "true" will
# - restart the WireGuard interface as the name suggests in case of changes
# - cause a short VPN connection interruption when changes are applied
# - make sure that network routes are properly reloaded
#
# So it depends a little bit on your setup which option works best. If you
# don't have an overly complicated routing that changes very often or at all
# using "false" here is most properly good enough for you. E.g. if you just
# want to connect a few servers via VPN and it normally stays this way.
#
# If you have a more dynamic routing setup then setting this to "true" might be
# the safest way to go. Also if you want to avoid the possibility creating some
# hard to detect side effects this option should be considered.
# If using netplan to configure WireGuard interfaces this option should be set
# to "true" if netplan configuration should be applied, otherwise it will
# just be generated.
wireguard_interface_restart: false

# Normally the role automatically creates a private key the very first time
# if there isn't already a WireGuard configuration. But this option allows
# to provide your own WireGuard private key if really needed. As this is of
# course a very sensitive value you might consider a tool like Ansible Vault
# to store it encrypted.
# wireguard_private_key:

# Set to "false" if package cache should not be updated (only relevant if
# the package manager in question supports this option)
wireguard_update_cache: "true"

# Normally the role installs and activates the wireguard kernel module where
# appropriate.  In some cases we might not be able load kernel modules, like
# unprivileged LXC guests.  If you set this to false you have to ensure
# the wireguard module is available in the kernel!
wireguard_install_kernel_module: true
```

There are also a few Linux distribution specific settings:

```yaml
#######################################
# Settings only relevant for:
# - Ubuntu
# - elementary OS
#######################################

# DEPRECATED: Please use "wireguard_update_cache" instead.
# Set to "false" if package cache should not be updated.
wireguard_ubuntu_update_cache: "{{ wireguard_update_cache }}"

# Set package cache valid time
wireguard_ubuntu_cache_valid_time: "3600"

# Set to "true" if netplan should be used to configure WireGuard interfaces
wireguard_ubuntu_use_netplan: false

#######################################
# Settings only relevant for CentOS 7
#######################################

# Set wireguard_centos7_installation_method to "kernel-plus"
# to use the kernel-plus kernel, which includes a built-in,
# signed WireGuard module.
#
# The default of "standard" will use the standard kernel and
# the ELRepo module for WireGuard.
wireguard_centos7_installation_method: "standard"

# Reboot host if necessary if the "kernel-plus" kernel is in use
wireguard_centos7_kernel_plus_reboot: true

# The default seconds to wait for machine to reboot and respond
# if "kernel-plus" is in use. Is only relevant if
# "wireguard_centos7_kernel_plus_reboot" is set to "true".
wireguard_centos7_kernel_plus_reboot_timeout: "600"

# Reboot host if necessary if the standard kernel is in use
wireguard_centos7_standard_reboot: true

# The default seconds to wait for machine to reboot and respond
# if "standard" kernel is in use. Is only relevant if
# "wireguard_centos7_standard_reboot" is set to "true".
wireguard_centos7_standard_reboot_timeout: "600"

#########################################
# Settings only relevant for RockyLinux 8
#########################################

# Set wireguard_rockylinux8_installation_method to "dkms"
# to build WireGuard module from source, with wireguard-dkms.
# This is required if you use a custom kernel and/or your arch
# is not x86_64.
#
# The default of "standard" will install the kernel module
# with kmod-wireguard from ELRepo.
wireguard_rockylinux8_installation_method: "standard"
```

Every host in `host_vars/` should configure at least one address via `wireguard_address` or `wireguard_addresses`. The `wireguard_address` can only contain one IPv4, thus it's recommended to use the `wireguard_addresses` variable that can contain an array of both IPv4 and IPv6 addresses.

```yaml
wireguard_addresses:
  - "10.8.0.101/24"
```

Of course all IP's should be in the same subnet like `/24` we see in the example above. If `wireguard_allowed_ips` is not set then the default values are IPs defined in `wireguard_address` and `wireguard_addresses` without the CIDR but instead with `/32` (IPv4) or `/128` (IPv6) which is basically a host route (have a look `templates/wg.conf.j2`). Let's see this example and let's assume you don't set `wireguard_allowed_ips` explicitly:

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = ....
ListenPort = 51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.101/32
Endpoint = controller01.p.domain.tld:51820
```

This is part of the WireGuard config from my workstation. It has the VPN IP `10.8.0.2` and we've a `/24` subnet in which all my WireGuard hosts are located. Also you can see we've a peer here that has the endpoint `controller01.p.domain.tld:51820`. When `wireguard_allowed_ips` is not explicitly set the Ansible template will add an `AllowedIPs` entry with the IP of that host plus `/32` or `/128`. In WireGuard this basically specifies the routing. The config above says: On my workstation with the IP `10.8.0.2` I want send all traffic to `10.8.0.101/32` to the endpoint `controller01.p.domain.tld:51820`. Now let's assume we set `wireguard_allowed_ips: "0.0.0.0/0"`. Then the resulting config looks like this.

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = ....
ListenPort = 51820

[Peer]
PublicKey = ....
AllowedIPs = 0.0.0.0/0
Endpoint = controller01.p.domain.tld:51820
```

Now this is basically the same as above BUT now the config says: I want to route EVERY traffic originating from my workstation to the endpoint `controller01.p.domain.tld:51820`. If that endpoint can handle the traffic is of course another thing and it's up to you how you configure the endpoint routing ;-)

You can specify further optional settings (they don't have a default and won't be set if not specified besides `wireguard_allowed_ips` as already mentioned) also per host in `host_vars/` (or in your Ansible hosts file if you like). The values for the following variables are just examples and no defaults (for more information and examples see [wg-quick.8](https://git.zx2c4.com/WireGuard/about/src/tools/man/wg-quick.8)):

```yaml
wireguard_allowed_ips: ""
wireguard_endpoint: "host1.domain.tld"
wireguard_persistent_keepalive: "30"
wireguard_dns: "1.1.1.1"
wireguard_fwmark: "1234"
wireguard_mtu: "1492"
wireguard_table: "5000"
wireguard_preup:
  - ...
wireguard_predown:
  - ...
wireguard_postup:
  - ...
wireguard_postdown:
  - ...
wireguard_save_config: "true"
```

`wireguard_(preup|predown|postup|postdown)` are specified as lists. Here are two examples:

```yaml
wireguard_postup:
  - iptables -t nat -A POSTROUTING -o ens12 -j MASQUERADE
  - iptables -A FORWARD -i %i -j ACCEPT
  - iptables -A FORWARD -o %i -j ACCEPT
```

```yaml
wireguard_preup:
  - echo 1 > /proc/sys/net/ipv4/ip_forward
  - ufw allow 51820/udp
```

The commands are executed in order as described in [wg-quick.8](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8).

Additionally one can add "unmanaged" peers. Those peers are not handled by Ansible and not part of the `vpn` Ansible host group e.g.:

```yaml
wireguard_unmanaged_peers:
  client.example.com:
    public_key: 5zsSBeZZ8P9pQaaJvY9RbELQulcwC5VBXaZ93egzOlI=
    # preshared_key: ... e.g. from ansible-vault?
    allowed_ips: 10.0.0.3/32
    endpoint: client.example.com:51820
    persistent_keepalive: 0
```

One of `wireguard_address` (deprecated) or `wireguard_addresses` (recommended) is required as already mentioned. It's the IPs of the interface name defined with `wireguard_interface` variable (`wg0` by default). Every host needs at least one unique VPN IP of course. If you don't set `wireguard_endpoint` the playbook will use the hostname defined in the `vpn` hosts group (the Ansible inventory hostname). If you set `wireguard_endpoint` to `""` (empty string) that peer won't have a endpoint. That means that this host can only access hosts that have a `wireguard_endpoint`. That's useful for clients that don't expose any services to the VPN and only want to access services on other hosts. So if you only define one host with `wireguard_endpoint` set and all other hosts have `wireguard_endpoint` set to `""` (empty string) that basically means you've only clients besides one which in that case is the WireGuard server. The third possibility is to set `wireguard_endpoint` to some hostname. E.g. if you have different hostnames for the private and public DNS of that host and need different DNS entries for that case setting `wireguard_endpoint` becomes handy. Take for example the IP above: `wireguard_address: "10.8.0.101"`. That's a private IP and I've created a DNS entry for that private IP like `host01.i.domain.tld` (`i` for internal in that case). For the public IP I've created a DNS entry like `host01.p.domain.tld` (`p` for public). The `wireguard_endpoint` needs to be a interface that the other members in the `vpn` group can connect to. So in that case I would set `wireguard_endpoint` to `host01.p.domain.tld` because WireGuard normally needs to be able to connect to the public IP of the other host(s).

## Example

Here is a litte example for what I use the playbook: I use WireGuard to setup a fully meshed VPN (every host can directly connect to every other host) and run my Kubernetes (K8s) cluster at Hetzner Cloud (but you should be able to use any hoster you want). So the important components like the K8s controller and worker nodes (which includes the pods) only communicate via encrypted WireGuard VPN. Also (as already mentioned) I've two clients. Both have `kubectl` installed and are able to talk to the internal Kubernetes API server by using WireGuard VPN. One of the two clients also exposes a WireGuard endpoint because the Postfix mailserver in the cloud and my internal Postfix needs to be able to talk to each other. I guess that's maybe a not so common use case for WireGuard :D But it shows what's possible. So let me explain the setup which might help you to use this Ansible role.

First, here is a part of my Ansible `hosts` file:

```ini
[vpn]
controller0[1:3].i.domain.tld
worker0[1:2].i.domain.tld
server.at.home.i.domain.tld
workstation.i.domain.tld

[k8s_controller]
controller0[1:3].i.domain.tld

[k8s_worker]
worker0[1:2].i.domain.tld
```

As you can see I've three groups here: `vpn` (all hosts on that will get WireGuard installed), `k8s_controller` (the Kubernetes controller nodes) and `k8s_worker` (the Kubernetes worker nodes). The `i` in the domainname is for `internal`. All the `i.domain.tld` DNS entries have a `A` record that points to the WireGuard IP that we define shortly for every host e.g.: `controller01.i.domain.tld. IN A 10.8.0.101`. The reason for that is that all Kubernetes components only binds and listen on the WireGuard interface in my setup. And since I need this internal IPs for all my Kubernetes components I specify the internal DNS entries in my Ansible `hosts` file. That way I can use the Ansible inventory hostnames and variables very easy in the playbooks and templates.

For the Kubernetes controller nodes I've defined the following host variables:

Ansible host file: `host_vars/controller01.i.domain.tld`

```yaml
---
wireguard_addresses:
  - "10.8.0.101/24"
wireguard_endpoint: "controller01.p.domain.tld"
ansible_host: "controller01.p.domain.tld"
ansible_python_interpreter: /usr/bin/python3
```

Ansible host file: `host_vars/controller02.i.domain.tld`:

```yaml
---
wireguard_addresses:
  - "10.8.0.102/24"
wireguard_endpoint: "controller02.p.domain.tld"
ansible_host: "controller02.p.domain.tld"
ansible_python_interpreter: /usr/bin/python3
```

Ansible host file: `host_vars/controller03.i.domain.tld`:

```yaml
---
wireguard_addresses:
  - "10.8.0.103/24"
wireguard_endpoint: "controller03.p.domain.tld"
ansible_host: "controller03.p.domain.tld"
ansible_python_interpreter: /usr/bin/python3
```

I've specified `ansible_python_interpreter` here for every node as the controller nodes use Ubuntu 18.04 which has Python 3 installed by default. `ansible_host` is set to the public DNS of that host. Ansible will use this hostname to connect to the host via SSH. I use the same value also for `wireguard_endpoint` because of the same reason. The WireGuard peers needs to connect to the other peers via a public IP (well at least via a IP that the WireGuard hosts can connect to - that could be of course also a internal IP if it works for you). IPs specified by `wireguard_address` or `wireguard_addresses` needs to be unique of course for every host.

For the Kubernetes worker I've defined the following variables:

Ansible host file: `host_vars/worker01.i.domain.tld`

```yaml
---
wireguard_addresses:
  - "10.8.0.111/24"
wireguard_endpoint: "worker01.p.domain.tld"
wireguard_persistent_keepalive: "30"
ansible_host: "worker01.p.domain.tld"
ansible_python_interpreter: /usr/bin/python3
```

Ansible host file: `host_vars/worker02.i.domain.tld`:

```yaml
---
wireguard_addresses:
  - "10.8.0.112/24"
wireguard_endpoint: "worker02.p.domain.tld"
wireguard_persistent_keepalive: "30"
ansible_host: "worker02.p.domain.tld"
ansible_python_interpreter: /usr/bin/python3
```

As you can see the variables are basically the same as the controller nodes have with one exception: `wireguard_persistent_keepalive: "30"`. My worker nodes (at Hetzner Cloud) and my internal server (my server at home) are connected because I've running Postfix at my cloud nodes and the external Postfix server forwards the received mails to my internal server (and vice versa). I needed the keepalive setting because from time to time the cloud instances and the internal server lost connection and this setting solved the problem. The reason for this is of course because my internal server is behind NAT and the firewall/router must keep the NAT/firewall mapping valid (NAT and Firewall Traversal Persistence).

For my internal server at home (connected via DSL router to the internet) we've this configuration:

```yaml
---
wireguard_addresses:
  - "10.8.0.1/24"
wireguard_endpoint: "server.at.home.p.domain.tld"
wireguard_persistent_keepalive: "30"
ansible_host: 192.168.2.254
ansible_port: 22
```

By default the SSH daemon is listening on a different port than 22 on all of my public nodes but internally I use `22` and that's the reason to set `ansible_port: 22` here. Also `ansible_host` is of course a internal IP for that host. The `wireguard_endpoint` value is a dynamic DNS entry. Since my IP at home isn't static I need to run a script every minute at my home server that checks if the IP has changed and if so adjusts my DNS record. I use OVH's DynHost feature to accomplish this but you can use and DynDNS provider you want of course. Also I forward incoming traffic on port `51820/UDP` to my internal server to allow incoming WireGuard traffic. IPs from `wireguard_address` and `wireguard_addresses` needs to be of course part of our WireGuard subnet.

And finally for my workstation (on which I run all `ansible-playbook` commands):

```yaml
wireguard_addresses:
  - "10.8.0.2/24"
wireguard_endpoint: ""
ansible_connection: local
ansible_become: false
```

As you can see `wireguard_endpoint: ""` is a empty string here. That means the Ansible role won't set an endpoint for my workstation. Since there is no need for the other hosts to connect to my workstation it doesn't makes sense to have a endpoint defined. So in this case I can access all hosts defined in the Ansible group `vpn` from my workstation but not the other way round. So the resulting WireGuard config for my workstation looks like this:

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = ....
ListenPort = 51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.101/32
Endpoint = controller01.p.domain.tld:51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.102/32
Endpoint = controller02.p.domain.tld:51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.103/32
Endpoint = controller03.p.domain.tld:51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.111/32
PersistentKeepalive = 30
Endpoint = worker01.p.domain.tld:51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.112/32
PersistentKeepalive = 30
Endpoint = worker02.p.domain.tld:51820

[Peer]
PublicKey = ....
AllowedIPs = 10.8.0.1/32
PersistentKeepalive = 30
Endpoint = server.at.home.p.domain.tld:51820
```

The other WireGuard config files (`wg0.conf` by default) looks similar but of course `[Interface]` includes the config of that specific host and the `[Peer]` entries lists the config of the other hosts.

## Example Playbooks

```yaml
- hosts: vpn
  roles:
    - githubixx.ansible_role_wireguard
```

```yaml
  hosts: vpn
  roles:
    -
      role: githubixx.ansible_role_wireguard
      tags: role-wireguard
```

## Example inventory using two different WireGuard interfaces on host "multi"

This is a complex example using yaml inventory format:

```yaml
vpn1:
  hosts:
    multi:
      wireguard_addresses:
        - "10.9.0.1/32"
      wireguard_allowed_ips: "10.9.0.1/32, 192.168.2.0/24"
      wireguard_endpoint: multi.example.com
    nated:
      wireguard_addresses:
        - "10.9.0.2/32"
      wireguard_allowed_ips: "10.9.0.2/32, 192.168.3.0/24"
      wireguard_persistent_keepalive: 15
      wireguard_endpoint: nated.example.com
      wireguard_postup:
        - iptables -t nat -A POSTROUTING -o ens12 -j MASQUERADE
        - iptables -A FORWARD -i %i -j ACCEPT
        - iptables -A FORWARD -o %i -j ACCEPT
      wireguard_postdown:
        - iptables -t nat -D POSTROUTING -o ens12 -j MASQUERADE
        - iptables -D FORWARD -i %i -j ACCEPT
        - iptables -D FORWARD -o %i -j ACCEPT

vpn2:
  hosts:
    # Use a different name, and define ansible_host, to avoid mixing of vars without
    # needing to prefix vars with interface name.
    multi-wg1:
      ansible_host: multi
      wireguard_interface: wg1
      # when using several interface on one host, we must use different ports
      wireguard_port: 51821
      wireguard_addresses:
        - "10.9.1.1/32"
      wireguard_endpoint: multi.example.com
    another:
      wireguard_addresses:
        - "10.9.1.2/32"
      wireguard_endpoint: another.example.com
```

Sample playbooks for example above:

```yaml
- hosts: vpn1
  roles:
    - githubixx.ansible_role_wireguard
```

```yaml
- hosts: vpn2
  roles:
    - githubixx.ansible_role_wireguard
```

## Testing

This role has a small test setup that is created using [Molecule](https://github.com/ansible-community/molecule), libvirt (vagrant-libvirt) and QEMU/KVM. Please see my blog post [Testing Ansible roles with Molecule, libvirt (vagrant-libvirt) and QEMU/KVM](https://www.tauceti.blog/posts/testing-ansible-roles-with-molecule-libvirt-vagrant-qemu-kvm/) how to setup. The test configuration is [here](https://github.com/githubixx/ansible-role-wireguard/tree/master/molecule/default).

Afterwards molecule can be executed:

```bash
molecule converge
```

This will setup quite a few virtual machines (VM) with different supported Linux operating systems. To run a few tests:

```bash
molecule verify
```

To clean up run

```bash
molecule destroy
```

There is also a small [Molecule setup](https://github.com/githubixx/ansible-role-wireguard/tree/master/molecule/single-server) that mimics a central WireGuard server with a few clients:

```bash
molecule converge -s single-server
```

## License

[GNU General Public License v3.0 or later](https://spdx.org/licenses/GPL-3.0-or-later.html)

## Author Information

[http://www.tauceti.blog](http://www.tauceti.blog)
