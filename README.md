# 128t-wireguard

Example implementation of a Wireguard gateway integrated with 128T.

More information about Wireguard can be found at https://www.wireguard.com/ .

## 128T Wireguard Setup

This walks step by step through the setup of the Wireguard peer on a 128T router host.

### Installation on the 128T host

Wireguard offers a RPM package for CentOS, however it has a dependency conflict with 128T (`elfutils`). You can easily compile it from source using some slight modifications of this procedure: https://www.wireguard.com/compilation/

#### 1- Install build dependencies
```
sudo yum install --enablerepo=updates kernel-devel-$(uname -r) pkgconfig "@Development Tools"
```

#### 2- Grab the code
```
git clone https://git.zx2c4.com/wireguard-linux-compat
git clone https://git.zx2c4.com/wireguard-tools
```

#### 3- Compile and install the module
```
$ make -C wireguard-linux-compat/src -j$(nproc)
$ sudo make -C wireguard-linux-compat/src install
```
#### 4- Compile and install the `wg` tool
```
$ make -C wireguard-tools/src -j$(nproc)
$ sudo make -C wireguard-tools/src install
```

### 128T configuration
This configuration assumes a network-interface already exists, with `128.128.128.128` as it's address. Packets coming from Wireguard peers will arrive to this address, and a service will route them in to a KNI.

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
Give `Work-From-Home` access to the required private services (this assume the private services already have routes configured).
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
The host KNI is configured to operate in an isolated network namespace called `wireguard`. Packets emerging from the Wireguard Linux interface will be routed back to this KNI, so the `Work-From-Home` tenant is configured.
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

        address    128.128.128.129
            ip-address     128.128.128.129
            prefix-length  31
            gateway        128.128.128.128
        exit
    exit
exit
```
```
service-route                       static-Wireguard
    name          static-Wireguard
    service-name  Wireguard

    next-hop      node1 wg-net
        node-name   node1
        interface   wg-net
        gateway-ip  128.128.128.128
    exit
exit
```

#### 5- Set up namespace
Someone deploying this would most likely want to use the `init` script contained in this repo. However for completeness in showing how to do the setup manually, the following sets up the namespace and moves the KNI `wg-dev` interface to it.
```
sudo ip netns add wireguard
sudo ip link set wg-dev netns wireguard
sudo ip netns exec wireguard ip address add 128.128.128.128/31 dev wg-dev
sudo ip netns exec wireguard ip link set wg-dev up
sudo ip netns exec wireguard ip route add default via 128.128.128.129 dev wg-dev
sudo ip netns exec wireguard sysctl -w -q net.ipv4.ip_forward=1
```

### Wireguard Configuration
Wireguard will be configured to use `10.10.128.1` as it's interface, and peers will be given remaining addresses from `10.10.128.0/24`.

#### 1- Set up Wireguard interface
Add the Wireguard interface to the namespace, and configure it with an address and network mask.
```
sudo ip netns exec wireguard ip link add wg0 type wireguard
sudo ip netns exec wireguard ip addr add 10.10.128.1/24 dev wg0
sudo ip netns exec wireguard ip link set up dev wg0
```

#### 2- Create keys and peer config files
Wireguard has a lot of flexibility in the ways in which it can be configured. For more information see https://www.wireguard.com/quickstart/ and `man wg`. For this we'll set up a conf and key files containing requisite configuration of and keys for the 128T Wireguard, and it's peers (`my-laptop` and `my-mobile`).
```
sudo mkdir /var/lib/128technology/plugins/wg
cd /var/lib/128technology/plugins/wg
sudo touch 128t.conf && sudo touch 128t.priv && sudo touch 128t.pub
sudo touch my-laptop.conf && sudo touch my-laptop.priv && sudo touch my-laptop.pub
sudo touch my-mobile.conf && sudo touch my-mobile.priv && sudo touch my-mobile.pub
sudo chmod 600 /var/lib/128technology/plugins/wg/*
```
Generate a new private/private key for each peer.
*Note*: best security practice would be to have each device generate keys on their own, and only transmit the public key between peers. This does not follow this best practice.
```
sudo wg genkey > /var/lib/128technology/plugins/wg/128t.priv
sudo wg pubkey < /var/lib/128technology/plugins/wg/128t.priv > /var/lib/128technology/plugins/wg/128t.pub
sudo wg genkey > /var/lib/128technology/plugins/wg/my-laptop.priv
sudo wg pubkey < /var/lib/128technology/plugins/wg/my-laptop.priv > /var/lib/128technology/plugins/wg/my-laptop.pub
sudo wg genkey > /var/lib/128technology/plugins/wg/my-mobile.priv
sudo wg pubkey < /var/lib/128technology/plugins/wg/my-mobile.priv > /var/lib/128technology/plugins/wg/my-mobile.pub
```
#### 3- Set up 128T Wireguard peer config
Set up the 128T peer `/var/lib/128technology/plugins/wg/128t.conf` file with settings for it's interface, and the peers.
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
#### 4- Set up remote peer configs
Set up the laptop peer `/var/lib/128technology/plugins/wg/my-laptop.conf` file with settings for it's interface, and the 128T peer.
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
Set up the mobile peer `/var/lib/128technology/plugins/wg/my-mobile.conf` file with settings for it's interface, and the 128T peer.
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
#### 5- Apply configuration to the Wireguard interface
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
