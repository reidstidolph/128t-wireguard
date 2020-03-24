# 128t-wireguard

Example implementation of a Wireguard gateway integrated with 128T.

## 128T Wireguard Setup

https://www.wireguard.com/compilation/

```
sudo yum install --enablerepo=updates kernel-devel-$(uname -r) pkgconfig "@Development Tools"
```

```
git clone https://git.zx2c4.com/wireguard-linux-compat
git clone https://git.zx2c4.com/wireguard-tools
```

```
$ make -C wireguard-linux-compat/src -j$(nproc)
$ sudo make -C wireguard-linux-compat/src install
```

```
$ make -C wireguard-tools/src -j$(nproc)
$ sudo make -C wireguard-tools/src install
```


