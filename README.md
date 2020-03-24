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

```
service            wireguard
    name                  wireguard
    description           "Wireguard VPN service"
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

```
device-interface  wg-dev
    name          wg-dev
    description   "Wireguard service function"
    type          host
    network-namespace  wireguard
    enabled        true

    network-interface  wg-net
        name       wg-net
        tenant     wfh.field-eng.corp

        address    128.128.128.129
            ip-address     128.128.128.129
            prefix-length  31
            gateway        128.128.128.128
        exit
    exit
exit
```
```
service-route                       static-wireguard
    name          static-wireguard
    service-name  wireguard

    next-hop      node1 wg-net
        node-name   node1
        interface   wg-net
        gateway-ip  128.128.128.128
    exit
exit
```

### Wireguard Configuration

```
sudo mkdir /var/lib/128technology/plugins/wg
sudo ip netns exec wireguard wg setconf wg0 /var/lib/128technology/plugins/wg/server.conf
sudo ip netns exec wireguard wg
```

## Remote Device Setup
