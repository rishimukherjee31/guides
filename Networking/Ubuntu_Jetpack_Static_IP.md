# Setting a Static IP on Jetson Orin Nano Super (JetPack 7.2)

This document will walk you through assigning a static IP address (`192.168.210.83`) to an NVIDIA Jetson Orin Nano Super Developer Kit running JetPack 7.2 (Jetson Linux 39.2, Ubuntu 24.04, 64-bit). A static IP means the Jetson keeps the same address every time it boots, which makes it reliable to reach over SSH and to use as a fixed node in a ROS2 network. The configuration is written as a netplan file that can be downloaded onto the Jetson from a URL, so the same setup can be repeated on a fresh flash or on additional devices.

This guide assumes JetPack 7.2 is already flashed and the Jetson has completed its first-boot setup. It is a companion to the ROS2 Jazzy installation guide, but does not depend on ROS2 being installed.

> **NOTE:** The file in this guide assigns the address and nothing else. It assumes a `192.168.210.0/24` network (subnet mask 255.255.255.0). It does **not** set a default gateway or DNS servers, so the Jetson can talk to other devices on `192.168.210.x` but will not be able to reach the internet (for example, `apt install` and `git clone` will fail). If you are following the ROS2 Jazzy guide, complete that installation first, while the Jetson still has internet access. If you need internet access later, see [Adding a Gateway and DNS (optional)](#adding-a-gateway-and-dns-optional). Choose an address that your router will not hand out to another device. Either pick one outside the router's DHCP range, or create a DHCP reservation for `192.168.xxx.xx` in the router's settings. Two devices with the same IP address cause intermittent, hard-to-diagnose connection drops.

## Contents

- [What's Included](#whats-included)
- [Check Your Network](#check-your-network)
- [Edit the Configuration](#edit-the-configuration)
- [Host the Netplan File](#host-the-netplan-file)
- [Download and Install](#download-and-install)
- [Apply the Configuration](#apply-the-configuration)
- [Verify the Configuration](#verify-the-configuration)
- [Adding a Gateway and DNS (optional)](#adding-a-gateway-and-dns-optional)
- [Rolling Back](#rolling-back)
- [Troubleshooting](#troubleshooting)
- [ROS2 Networking Tips](#ros2-networking-tips)

<br>

## What's Included

| File | Purpose |
|------|---------|
| `99-static-ip.yaml` | The netplan configuration that sets the static IP address. |

Netplan is Ubuntu's network configuration system. You describe the network you want in a YAML file in `/etc/netplan/`, and netplan turns that into configuration for the underlying network service (either NetworkManager or systemd-networkd). Files in that folder are read in filename order, and later files override earlier ones. The file here is named `99-static-ip.yaml` so that it is read last and takes priority over anything else on the system.

<br>

## Check Your Network

Before changing anything, find out how the Jetson is currently connected. This tells you the name of the network interface and which network service is managing it. The Jetson's wired port is usually **not** called `eth0`. It typically has a name like `enP8p1s0`, and Wi-Fi typically looks like `wlP1p1s0`:

```bash
ip -br a                 # list interfaces and their current IP addresses
ip route                 # shows the current default gateway (the "default via" line)
nmcli device status      # shows which interface is connected and how it is managed
```

Write down two things:

- **Interface name** of the connection you want to make static (for example, `enP8p1s0`).
- **Subnet** from the address shown in `ip -br a` (for example, `192.168.210.x/24`). Your static IP must be in this same subnet as the other devices you want to reach.

The `ip route` output also shows your current gateway (the `default via ...` line). You do not need it for this configuration, but note it down in case you want to add internet access later.

Then check which service is managing the network. This decides which `renderer` value to use in the YAML file:

```bash
systemctl is-active NetworkManager    # "active" means NetworkManager is in use
```

If the output is `active`, keep `renderer: NetworkManager`. Otherwise, change it to `renderer: networkd`.

<br>

## Edit the Configuration

Open `99-static-ip.yaml` and check each value against what you found in the previous step:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    wired:
      renderer: networkd
      match:
        name: "en*"
      dhcp4: false
      dhcp6: false
      ignore-carrier: true
      addresses:
        - 192.168.210.83/24
```

What each part does:

- `renderer` selects the network service netplan hands the configuration to. Use `NetworkManager` if it is running, otherwise `networkd`.
- `match: name: "en*"` applies the configuration to any interface whose name starts with `en`, which covers the wired port on the dev kit. If your Jetson has more than one wired interface, replace this with the exact name (for example, `name: "enP8p1s0"`) or match on the hardware address with `macaddress: "aa:bb:cc:dd:ee:ff"`.
- `dhcp4: false` turns off automatic addressing, so the address below is used instead.
- `addresses` is the static IP, written with its prefix length. `/24` is the same as a subnet mask of 255.255.255.0.

No `routes` or `nameservers` are set. The `/24` in the address already gives the Jetson a route to every device on `192.168.210.0/24`, which is all that is needed for SSH and ROS2 traffic between machines on that network.

> **NOTE:** YAML is sensitive to indentation. Use spaces only (never tabs), and keep the nesting exactly as shown.

<br>

## Host the Netplan File

For the download to be automated, the Jetson needs to be able to fetch the YAML file from a URL. Any location it can reach will work, for example:

- A raw file link in a GitHub repository (use the `raw.githubusercontent.com` address).
- An internal web server on your network.
- A file share or object storage bucket.

Test the link from the Jetson before installing anything:

```bash
curl -fsSL https://your-host/99-static-ip.yaml | head
```

You should see the YAML contents. If you see an HTML page or an error message, the link is not pointing at the raw file.

> **NOTE:** This file becomes your network configuration, and it is installed as root. Only use HTTPS, and only host the file somewhere that you control. Anyone who can edit the file at that URL can change the network settings of every device that downloads it.

<br>

## Download and Install

Download the file directly into the netplan folder, then set the ownership and permissions. Netplan requires that configuration files are not readable by other users, and prints warnings if they are:

```bash
sudo curl -fsSL https://your-host/99-static-ip.yaml -o /etc/netplan/99-static-ip.yaml
sudo chown root:root /etc/netplan/99-static-ip.yaml
sudo chmod 600 /etc/netplan/99-static-ip.yaml
```

Next, check that the file is valid. `netplan generate` reads the configuration and reports any errors without changing anything on the running system:

```bash
sudo netplan generate
```

If it prints nothing, the syntax is correct. If it prints an error, note the line number in the message and fix the file before continuing. The most common causes are indentation, a tab character, or a missing colon.

<br>

## Apply the Configuration

Apply the configuration with `netplan try` rather than `netplan apply`. It applies the new settings and then waits for you to confirm. If you lose your connection because of a mistake (for example, a wrong interface match), the previous configuration is restored automatically after 120 seconds:

```bash
sudo netplan try
```

When prompted, press Enter to keep the new configuration.

> **NOTE:** If you are connected over SSH, your session will drop when the IP address changes to `192.168.210.83`, and the `netplan try` prompt will time out and revert the change. To keep the change, run these commands from a local keyboard and monitor (or a serial console). If you are confident the file is correct and want to apply it over SSH, use `sudo netplan apply` instead, then reconnect to `192.168.210.83`.

<br>

## Verify the Configuration

Once the configuration is applied, confirm that everything works. Run each of these on the Jetson:

```bash
ip -br a                       # should show 192.168.210.83/24 on your interface
ip route                       # should show: 192.168.210.0/24 dev <interface> ... src 192.168.210.83
ping -c 3 <another device>     # any other device on 192.168.210.x
```

`ip route` will not show a `default via` line. That is expected with this configuration.

Then, from a **different computer** on the same network, confirm you can reach the Jetson:

```bash
ping 192.168.210.83
ssh <username>@192.168.210.83
```

Finally, reboot the Jetson and repeat the checks to confirm that the settings survive a restart:

```bash
sudo reboot
```

<br>

## Adding a Gateway and DNS (optional)

Without a default gateway, the Jetson cannot reach the internet. If you later need to install packages or download files, add the following to the interface section of the YAML file (indented to line up with `addresses`), then re-run `sudo netplan try`. Replace `192.168.210.1` with your router's address, which is shown on the `default via` line of `ip route` while the Jetson is still on DHCP:

```yaml
      routes:
        - to: default
          via: 192.168.210.1
      nameservers:
        addresses:
          - 192.168.210.1
          - 1.1.1.1
```

`routes` sets the default gateway, the router that traffic to other networks goes through. `nameservers` lists the DNS servers used to turn names like `ros.org` into addresses. To check that it worked, run `ping -c 3 1.1.1.1` (routing) and `ping -c 3 ros.org` (DNS).

<br>

## Rolling Back

If you need to return to automatic (DHCP) addressing, remove the file and reapply:

```bash
sudo rm /etc/netplan/99-static-ip.yaml
sudo netplan apply
```

The Jetson will go back to whatever configuration is defined by the other files in `/etc/netplan/`, or to the profile saved in NetworkManager. If you cannot reach the Jetson over the network, connect a keyboard and monitor (or a serial console) and run the same commands locally.

<br>

## Troubleshooting

- **The Jetson has no internet after applying.** This is expected, because the configuration does not set a gateway or DNS. See [Adding a Gateway and DNS (optional)](#adding-a-gateway-and-dns-optional).
- **The Jetson cannot reach other devices.** Check that the ethernet cable is connected and that `192.168.210.83` is inside the same subnet as the other devices.
- **`netplan generate` reports an error.** Read the line number in the message. It is almost always indentation, a tab character, or a missing colon in the YAML.
- **The Jetson still has the old DHCP address after applying.** NetworkManager may have a saved profile (for example, "Wired connection 1") that is taking priority. List the profiles with `nmcli connection show`, and check which one is active on your interface with `nmcli device status`. Delete or disable the old DHCP profile if it is conflicting, then reapply.
- **The configuration has no effect at all.** The `renderer` may not match the service that is running. Check `systemctl is-active NetworkManager` and set `renderer` to `NetworkManager` or `networkd` accordingly.
- **"Permissions are too open" warning from netplan.** The file in `/etc/netplan/` must be `600`. Fix it with `sudo chmod 600 /etc/netplan/99-static-ip.yaml`.
- **The address works, but only some devices can reach the Jetson.** Another device may already be using `192.168.210.83`. Check your router's list of connected devices, or run `arping -D -I <interface> 192.168.210.83` from another Linux machine to test for duplicates.
- **The interface name does not match.** If `ip -br a` shows a wired interface that does not begin with `en`, change the `match` name in the YAML to the correct name.
- **Everything works on ethernet but nothing works after switching to Wi-Fi.** The YAML file only configures wired interfaces. Wi-Fi requires its own `wifis:` section with the network name and password, which is not included here.

<br>

## ROS2 Networking Tips

These recommendations apply if the Jetson is part of a ROS2 network with other machines.

- **Use the same `ROS_DOMAIN_ID` on every machine that should communicate.** Machines on the same network with different domain IDs will not see each other's topics. Pick a number between 0 and 101 and add it to `~/.bashrc` on each machine:

```bash
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
```

- **Keep all machines on the same subnet.** Default DDS discovery uses multicast, which is normally not forwarded between subnets. All machines should have addresses in `192.168.210.0/24`.
- **Check for multicast blocking, especially on Wi-Fi.** If ROS2 nodes discover each other over ethernet but not over Wi-Fi, the access point is most likely blocking multicast traffic. Use a wired connection for the Jetson where possible.
- **Check the firewall.** If `ufw` is enabled (`sudo ufw status`), it can block DDS traffic. Allow traffic from your network with:

```bash
sudo ufw allow from 192.168.210.0/24
```

- **Pin DDS to the correct network interface if the Jetson has more than one active.** If both ethernet and Wi-Fi are connected, DDS may choose the wrong one. With Cyclone DDS (`rmw_cyclonedds_cpp`), you can pin it to one interface. Replace `enP8p1s0` with your interface name:

```bash
echo 'export CYCLONEDDS_URI="<CycloneDDS><Domain><General><Interfaces><NetworkInterface name=\"enP8p1s0\"/></Interfaces></General></Domain></CycloneDDS>"' >> ~/.bashrc
```

- **Test communication between machines.** With the Jetson running `ros2 run demo_nodes_cpp talker` and another machine (same `ROS_DOMAIN_ID`) running `ros2 run demo_nodes_py listener`, the listener should print the Jetson's messages. If it does not, work through the multicast, firewall, and domain ID checks above.
