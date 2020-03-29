# 128t-wireguard

Example implementation of a Wireguard gateway integrated with 128T.

More information about Wireguard can be found at https://www.wireguard.com/ .

## 128T Wireguard Setup

This walks step by step through the setup of the Wireguard peer on a 128T router host. Wireguard peers will be allocated from a `10.10.128.0/24` private network. The 128T router wireguard peer will be `10.10.128.1`, and additional devices will be allocated private addresses from `10.10.128.2-254`. Sessions arriving from devices will be assigned a `Work-From-Home` tenant as they are sent from the Wireguard interface back into the 128T forwarding plane.

### Installation on the 128T host

Wireguard offers a RPM package for CentOS, and you can attempt to install it using instructions here: https://www.wireguard.com/install/.

However you may run into a dependency conflict with 128T (`elfutils`). You can easily compile it from source using some slight modifications of this procedure: https://www.wireguard.com/compilation/

#### 1- Install build dependencies
Install build dependencies on a target 128T host.
```
sudo yum install --enablerepo=updates kernel-devel-$(uname -r) pkgconfig "@Development Tools"
```

#### 2- Grab the code
Use `git` to download the code.
```
git clone https://git.zx2c4.com/wireguard-linux-compat
git clone https://git.zx2c4.com/wireguard-tools
```

#### 3- Compile and install the module
```
make -C wireguard-linux-compat/src -j$(nproc)
sudo make -C wireguard-linux-compat/src install
```

#### 4- Compile and install the `wg` tool
```
make -C wireguard-tools/src -j$(nproc)
sudo make -C wireguard-tools/src install
```

### Add plugin scripts
A KNI will be used receive Wireguard packets from the forwarding plane, and send clear packets back in to the forwarding plane. In preparation for the configuration of the KNI to be applied, set up some network plugin scripts.

#### 1- Create directories
```
sudo mkdir -p /var/lib/128technology/plugins/wg
sudo mkdir -p /etc/128technology/plugins/network-scripts/host/wg-dev
```

#### 2- Place `init` and `shutdown` scripts
Place the `init` and `shutdown` scripts contained in this repo in `/etc/128technology/plugins/network-scripts/host/wg-dev`, and make them executable.
```
sudo chmod+x /etc/128technology/plugins/network-scripts/host/wg-dev/*
```

#### `init` script explanation
You DO NOT need to run these commands, as they are handled by the `init` script. However for the sake of explaining what the script is doing, the following explains what the script is doing as if you were to do it manually.
```
# create the namespace
sudo ip netns add wireguard

# move the KNI to the namespace and configure it
sudo ip link set wg-dev netns wireguard
sudo ip netns exec wireguard ip address add 128.128.128.128/31 dev wg-dev
sudo ip netns exec wireguard ip link set wg-dev up

# add the Wireguard interface to the namespace
sudo ip netns exec wireguard ip link add wg0 type wireguard
sudo ip netns exec wireguard ip addr add 10.10.128.1/24 dev wg0
sudo ip netns exec wireguard ip link set up dev wg0

# enable IP forwarding in the namespace and set route in to forwarding plane
sudo ip netns exec wireguard sysctl -w -q net.ipv4.ip_forward=1
sudo ip netns exec wireguard ip route add default via 128.128.128.129 dev wg-dev
```

### 128T configuration
This configuration assumes a network-interface already exists, and `128.128.128.128` routes to it. Packets coming from Wireguard peers will arrive to this address, and a service will route them in to a KNI.

#### 1- Wireguard Service
So that packets arriving from peers anywhere will match this service, it is set to `public`.
```
service            Wireguard
    name                  Wireguard
    description           "Public Wireguard service"
    scope                 public

    transport             udp
        protocol    udp

        port-range  12800
            start-port  12800
            end-port    12800
        exit
    exit
    address               128.128.128.128

    share-service-routes  false
exit
```

#### 2- Work From Home Tenant
Sessions coming from remote endpoints must be given a network tenant. Existing tenancy could be used, but in this case a new top level tenant is established called `Work-From-Home`.
```
tenant             Work-From-Home
    name         Work-From-Home
    description  "Endpoints for users working from home"
exit
```

#### 3- Access to Private Services
Give `Work-From-Home` access to the required private services (this assumes the private services already have routes configured).
```
service            Corp-Net
    name                  Corp-Net
    description           "Private corporate service"
    scope                 private
    security              corp-sec
    address               172.16.128.0/24

    access-policy         Work-From-Home
        source  Work-From-Home
    exit
exit
```

#### 4- KNI Interface
The host KNI is configured to operate in an isolated network namespace called `wireguard`. Packets emerging from the Wireguard Linux interface will be routed back to this KNI, so the `Work-From-Home` tenant is configured. Addressing for the interface is set to link-local, as is common for network plugins.
```
device-interface  wg-dev
    name          wg-dev
    description   "Wireguard service function"
    type          host
    network-namespace  wireguard
    enabled        true

    network-interface  wg-net
        name       wg-net
        tenant     Work-From-Home

        address    169.254.129.64
            ip-address     169.254.129.64
            prefix-length  31
            gateway        169.254.129.65
        exit
    exit
exit
```
#### 5- Add a route
Route the Wireguard service in to the KNI, with a destination NAT to the KNI.
```
service-route                       static-Wireguard
    name          static-Wireguard
    service-name  Wireguard
    nat-target    169.254.129.65

    next-hop      node1 wg-net
        node-name   node1
        interface   wg-net
        gateway-ip  169.254.129.65
    exit
exit
```

### Wireguard Configuration
Wireguard will be configured to use `10.10.128.1` as its interface, and peers will be given remaining addresses from `10.10.128.0/24`.

#### 1- Create keys and peer config files
Wireguard has a lot of flexibility in the ways in which it can be configured. For more information see https://www.wireguard.com/quickstart/ and `man wg`. For this we'll set up a conf and key files containing requisite configuration of and keys for the 128T Wireguard, and its peers (`my-laptop` and `my-mobile`).
```
cd /var/lib/128technology/plugins/wg
sudo touch 128t.conf && sudo touch 128t.priv && sudo touch 128t.pub
sudo touch my-laptop.conf && sudo touch my-laptop.priv && sudo touch my-laptop.pub
sudo touch my-mobile.conf && sudo touch my-mobile.priv && sudo touch my-mobile.pub
sudo chmod 600 /var/lib/128technology/plugins/wg/*
```
Generate a new private/private key for each peer.

*Note*: best security practice would be to have each device generate keys on their own, and only transmit the public key between peers. This does not follow this best practice. These must be run as `root`, and are for the purpose of demonstration.
```
wg genkey > /var/lib/128technology/plugins/wg/128t.priv
wg pubkey < /var/lib/128technology/plugins/wg/128t.priv > /var/lib/128technology/plugins/wg/128t.pub
wg genkey > /var/lib/128technology/plugins/wg/my-laptop.priv
wg pubkey < /var/lib/128technology/plugins/wg/my-laptop.priv > /var/lib/128technology/plugins/wg/my-laptop.pub
wg genkey > /var/lib/128technology/plugins/wg/my-mobile.priv
wg pubkey < /var/lib/128technology/plugins/wg/my-mobile.priv > /var/lib/128technology/plugins/wg/my-mobile.pub
```
#### 2- Set up 128T Wireguard peer config
Set up the 128T peer `/var/lib/128technology/plugins/wg/128t.conf` file with settings for its interface, and the peers.
```
[Interface]
ListenPort = 12800
PrivateKey = <128t private key string>

[Peer]
# My laptop peer
AllowedIPs = 10.10.128.2/32
PublicKey = <my-laptop public key string>

[Peer]
# My mobile peer
AllowedIPs = 10.10.128.3/32
PublicKey = <my-mobile public key string>
```
#### 3- Set up remote peer configs
Set up the laptop peer `/var/lib/128technology/plugins/wg/my-laptop.conf` file with settings for its interface, and the 128T peer. Note the `AllowedIPs` setting corresponds with the private service address we want to send to the 128T Wireguard peer. You can customize this setting for your environment, to control what the remote device sends to the 128T peer, and what it sends directly out its default route.
```
[Interface]
Address = 10.10.128.2/32
PrivateKey = <my-laptop private key string>
DNS = 1.1.1.1

[Peer]
PublicKey = <128T public key string>
AllowedIPs = 172.16.128.0/24
Endpoint = 128.128.128.128:12800
```
Set up the mobile peer `/var/lib/128technology/plugins/wg/my-mobile.conf` file with settings for its interface, and the 128T peer. Note the `AllowedIPs` setting corresponds with the private service address we want to send to the 128T Wireguard peer. You can customize this setting for your environment, to control what the remote device sends to the 128T peer, and what it sends directly out its default route.
```
[Interface]
Address = 10.10.128.3/32
PrivateKey = <my-mobile private key string>
DNS = 1.1.1.1

[Peer]
PublicKey = <128T public key string>
AllowedIPs = 172.16.128.0/24
Endpoint = 128.128.128.128:12800
```
#### 4- Apply configuration to the Wireguard interface
Apply the 128T Wireguard configuration to the interface within the namespace.
```
sudo ip netns exec wireguard wg setconf wg0 /var/lib/128technology/plugins/wg/128t.conf
```
You can verify the configuration is applied by inspecting the output of `wg`.
```
sudo ip netns exec wireguard wg
```
If you make changes to the config, such as adding new peers, you can have Wireguard re-evaluate your config.
```
sudo ip netns exec wireguard wg syncconf wg0 /var/lib/128technology/plugins/wg/128t.conf
```

## Remote Device Setup
To set up the remote devices, copy the conf settings to the devices. For Android and iOS mobile devices, you can generate QR codes.
```
sudo yum install qrencode
sudo qrencode -t ansiutf8 < /var/lib/128technology/plugins/wg/my-mobile.conf
```

## Plugin Scripts
To automate the steps for setting up the namespace and interfaces, this repo contains a `init` and `shutdown` plugin script. Copy them to `/etc/128technology/plugins/network-scripts/host/wg-dev/` on your 128T host, and the interfaces will be automatically added and removed as 128T is started/stopped.
