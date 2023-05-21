ansible-router
==============

I use this role to configure my home router, a [PC Engines APU2][apu2] running
OpenBSD. Notable functionality includes:

- Dual-stack IPv4/IPv6
- Wired and wireless interfaces bridged using [veb(4)][]
- Local DNS resolver with local records and domain blocking
- A pf configuration with NAT and simple firewall rules

This role is designed only for my personal use and contains various hardcoded
assumptions and limitations. Use at your own risk.

The machine configured here has three NICs, `em0`, `em1`, and `em2`, and
a wireless interface, `athn0`. `em0` is the egress, and `em1`, `em2`, and
`athn0` are all connected via a [veb(4)][] bridge with a vport(4) interface
named `vport0`.

Requirements
------------

The target system must be prepared with a Python installation, SSH access,
a suitable [doas(1)][] configuration, and any other prerequisites required
for use with Ansible.

Role Variables
--------------

- `hostname`: The router's hostname.
- `search_domain`: FQDN to use for DNS search.
- `root_email`: Email address to use for the root user and various aliases.
- `smtpd_secrets`: Any variable definitions containing secrets for inclusion
  in `/etc/smtpd.conf`. See the sample playbook below for an example.
- `smtp_host`: SMTP host specification for smtp relay. See the example below
  and [smtpd.conf(5)][].
- `mail_from`: SMTP `MAIL FROM` address to use when sending mail.
- `root_from_addr`: "From" address to set in .mailrc for root user.
- `wifi`: Wireless network configuration parameters. See [hostname.if(5)][] and
  [ifconfig(8)][].
	- `key`: WPA key
	- `nwid`: Network ID
	- `chan`: Channel
	- `enabled`: Whether to bring the interface up. Optional, defaults to
	  `true`.
- `vport0`: Configuration parameters for the subnet assigned to the vport
  interface attached to the [veb(4)][] bridge.
  	- `ipv4`: IPv4 configuration parameters.
		- `addr`: The interface's IPv4 address.
		- `subnet`: IPv4 subnet address.
		- `netmask`: The subnet's netmask.
		- `broadcast_addr`: The subnet's broadcast address.
		- `cidr`: The subnet expressed in CIDR notation.
	- `ipv6`: IPv6 configuration parameters.
		- `ula`: A unique local address for the vport0 interface
		  conforming to [RFC 4193][].
		- `prefixlen`: The ULA subnet's prefix length.
		- `cidr`: The ULA subnet expressed in CIDR notation.
- `dhcpd`: dhcpd(8) configuration.
	- `dynamic`: Range definition for dynamically assigned IP addresses.
		- `start`: The lowest address in the range.
		- `end`: The highest address in the range.
	- `static`: A list of static IP address assignment configurations. Each
	  configuration requires the following variables:
		- `name`: Hostname.
		- `hw_addr`: MAC address.
		- `ip_addr`: IP address to assign.
- `static_port_ips`: A list of IP addresses for which static ports should be
  used in network address translation, required for some games to function
  properly.
- `local_dns_records`: A list of configurations for DNS records to be served by
  unbound(8) on the local network. Parameters include:
	- `type`: A or AAAA.
	- `domain`: The domain for the record.
	- `addr`: The IP address for the record.
	- `ptr`: Whether to create a corresponding PTR record.
	  Optional, defaults to `true`.
- `nameservers`: A list of IP addresses of upstream DNS servers that
  `unbound(8)` should forward requests to. An item may be specified as either
  a string or a dict. If a string is specified, it should be a plain IP
  address, and port 853 (for DNS over TLS) is assumed.  If a dict is specified,
  it must have two keys:
	- `addr`: The server's IP address.
	- `port`: The server's port number.
- `blocked_domains`: A list of domains to block via unbound(8).

Example Playbook
----------------

This playbook configures a machine named router.example.com with internal
subnets at 192.168.1.0/24 and fdfd:64a6:2917::/64.

```yaml
- hosts: servers
  vars:
    hostname: router.example.com
    search_domain: example.com.
    root_email: "user@example.com"

    # mail_password should be stored in a vault.
    smtpd_secrets: mymailauth:user@example.com:{{ mail_password }}
    smtp_host: "smtp+tls://mymailauth@smtp.example.com:587"
    mail_from: "user@example.com"
    root_from_addr: "user@example.com"

    wifi:
      # Key should be stored in a vault.
      key: "{{ my_wifi_password }}"
      nwid: myssid
      chan: 132

    vport0:
      ipv4:
        addr: 192.168.1.1
        subnet: 192.168.1.0
        netmask: 255.255.255.0
        broadcast_addr: 192.168.1.255
        cidr: 192.168.1.0/24
      ipv6:
        # This should be a unique RFC 4193-compliant subnet.
        ula: fdfd:64a6:2917::1
        prefixlen: 64
        cidr: fdfd:64a6:2917::/64

    dhcpd:
      # Hand out 192.168.1.50-192.168.1.100 (inclusive) as dynamic
      # addresses.
      dynamic:
        start: 192.168.1.50
        end: 192.168.1.100

      # Assign some static addresses based on MAC addresses.
      static:
        - name: wireless-access-point
          hw_addr: 28:10:89:8F:34:8E
          ip_addr: 192.168.1.2
        - name: desktop
          hw_addr: 16:F5:E4:0B:19:3C
          ip_addr: 192.168.1.3

    # Don't rewrite the source port on packets when natting to desktop.
    static_port_ips:
      - 192.168.1.3

    # Set up some local DNS records so we can find our router on the LAN.
    local_dns_records:
      - type: A
        domain: router.example.com
        addr: "{{ vport0.ipv4.addr }}"
      - type: AAAA
        domain: router.example.com
        addr: "{{ vport0.ipv6.ula }}"

    # Configure upstream DNS servers to which unbound should forward
    # requests.
    nameservers:
      # Verbose format -- allows port to be specified.
      - addr: 1.1.1.1
        port: 853

      # Simple format: port 853 is assumed.
      - 8.8.8.8
      - 2001:4860:4860::8888

    # Tell unbound to refuse queries for certain domains.
    blocked_domains:
      - blocked.domain.example
      - another-blocked-domain.example

  roles:
     - router
```

License
-------

ISC

Author Information
------------------

Benjamin Linskey
<contact@linskey.org>

[apu2]: https://www.pcengines.ch/apu2.htm
[veb(4)]: https://man.openbsd.org/veb.4
[doas(1)]: https://man.openbsd.org/man1/doas.1
[smtpd.conf(5)]: https://man.openbsd.org/smtpd.conf
[hostname.if(5)]: https://man.openbsd.org/hostname.if
[ifconfig(8)]: https://man.openbsd.org/ifconfig
[RFC 4193]: https://www.rfc-editor.org/rfc/rfc4193
[dhcpd(8)]: https://man.openbsd.org/dhcpd
[unbound(8)]: https://man.openbsd.org/unbound
