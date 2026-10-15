# ansible-role-network

NetworkManager profiles and firewalld zones on openSUSE Leap, from the inventory. Built after the setup of the old
Fedora laptop: unknown networks get nothing in (default zone `drop`), the own LAN and Wi-Fi are in zone `work`
(mDNS for the driverless printer, syncthing, ssh), LAN and Wi-Fi share one MAC and thus one DHCP reservation, so
switching between cable and Wi-Fi (or using both) is seamless.

1. firewalld (only if running): per zone in `firewalld.zones` exactly the listed services (missing ones are added,
   others removed - permanent and at once); `firewalld.default_zone` if set. Zones not listed are not touched
   (e.g. `libvirt` with nfs from ansible-role-nfsserver).
2. `same_mac` → `/etc/NetworkManager/conf.d/99-sameMAC.conf`: this MAC for all Ethernet and Wi-Fi connections.
3. one profile per entry in `connections` → `/etc/NetworkManager/system-connections/ansible-<id>.nmconnection`
   (root, 0600, stable uuid from the id): Ethernet without interface binding (any cable or dock), Wi-Fi with the
   password in the file (system-wide, also before login), `cloned-mac-address=permanent`, no MAC randomization.
   Wi-Fi files are written without output/diff (password).
4. `nmcli connection reload` / `nmcli general reload conf` on changes.

**Active connections are not reconnected** (no network loss during the run): new profiles, zones and the MAC
apply at the next connect. Reconnect once (unplug, Wi-Fi off/on, or `nmcli connection up <id>`).

Other NetworkManager profiles are never changed or deleted - also not a profile you remove from `connections`
(delete its `ansible-<id>.nmconnection` by hand). If another profile has the same name as a managed one (e.g.
created by the installer for your Wi-Fi), the role warns - delete it by hand: `nmcli connection delete <uuid>`.

**Lockout:** with `default_zone: drop`, every interface without a managed connection (and without its own zone)
loses incoming SSH at once. Hosts you reach via SSH on such an interface (e.g. a VM) need another default zone.

**The only firewalld writer of the desktop playbook:** the service lists of all zones that need services come from
here (e.g. `syncthing` in the zone of your networks, `nfs` in zone `libvirt`); ansible-role-syncthing and
ansible-role-nfsserver don't touch the firewall. Zones not listed are left as they are. Warnings (shown as
failed + "...ignoring" and counted as `ignored` in the recap; the run continues): `network` not defined for the
host; a listed zone that nothing uses (not the default zone, no managed connection, no interfaces/sources);
another NetworkManager profile with the name of a managed one.

## Variables

```yaml
network:
  firewalld:
    default_zone: drop
    zones:
      work:
        services: [ssh, mdns, dhcpv6-client, syncthing]
  same_mac: "aa:bb:cc:dd:ee:ff"      # host_vars: the Wi-Fi MAC of this laptop
  connections:
    - id: dhcp
      type: ethernet
      zone: work
    - id: MyWifi
      type: wifi
      ssid: MyWifi                   # default: id
      key_mgmt: sae                  # sae (WPA3) or wpa-psk (WPA2)
      psk: "{{ vault_network_wifi_mywifi_psk }}"
      zone: work
```

A zone like `libvirt` needs its complete list, including the services the package brings (desired state: what
is not listed is removed), e.g. `[dhcp, dhcpv6, dns, ssh, tftp, nfs]`.

## Check

```bash
firewall-cmd --get-default-zone; firewall-cmd --get-active-zones
firewall-cmd --zone=work --list-services
nmcli -f NAME,UUID,TYPE,DEVICE connection show
ip link                              # LAN and Wi-Fi with the same MAC once connected
avahi-browse -rt _ipp._tcp           # printers via mDNS (package avahi-utils)
```

## License

Apache-2.0

Created with the help of AI
