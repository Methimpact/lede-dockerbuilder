# Dockerized OpenWrt image builder

[![Build Status](https://travis-ci.org/jandelgado/lede-dockerbuilder.svg?branch=master)](https://travis-ci.org/jandelgado/lede-dockerbuilder)

<!-- vim-markdown-toc GFM -->

* [What](#what)
    * [Note](#note)
* [Why](#why)
* [How](#how)
    * [Usage](#usage)
    * [Configuration file](#configuration-file)
    * [File system overlay](#file-system-overlay)
    * [Example directory structure](#example-directory-structure)
        * [Debugging](#debugging)
* [Examples](#examples)
    * [Building a x86_64 image and running it in qemu](#building-a-x86_64-image-and-running-it-in-qemu)
* [Building an OpenWrt snapshot release](#building-an-openwrt-snapshot-release)
* [Author](#author)
* [License](#license)

<!-- vim-markdown-toc -->

## What

Easily and quickly build OpenWrt custom images (e.g. for your embedded device our Raspberry PI) 
using a self-contained docker container and the
[OpenWrt](https://wiki.openwrt.org/doc/howto/obtain.firmware.generate) image
builder. On the builder host, Docker is the only requirement. Supports latest
OpenWrt release (18.06.x).

### Note

The OpenWrt-dockerbuilder uses pre-compiled packages to build the final image. 
Look [here](https://github.com/jandelgado/lede-dockercompiler) if you are looking 
for a docker images to compile OpenWrt completely from source.

## Why

* customized and optimized (size) images with your personal configurations
* full automatic image creation (could be run in CI)
* reproducable results
* easy configuration, fast build (in minutes)

## How

```
$ git clone https://github.com/jandelgado/lede-dockerbuilder.git
$ cd lede-dockerbuilder
$ ./builder.sh build example-nexx-wt3020.conf
```
Your custom images can now be found in the `output` diretory.

The `build` command will first build the docker image containing the actual image
builder. The resulting docker image is per default tagged with
`openwrt-imagebuilder:<Release>-<Target>-<Subtarget>`.  Afterwards a container,
which builds the actual OpenWrt image, is run. The final OpenWrt image will be
available in the `output/` directory.

### Usage
```
Dockerized LEDE/OpenWrt image builder.

Usage: ./builder.sh COMMAND CONFIGFILE [OPTIONS] 
  COMMAND is one of:
    build-docker-image- just build the docker image
    build             - build docker image, then start container and build the LEDE/OpenWrt image
    shell             - start shell in docker container
  CONFIGFILE          - configuraton file to use

  OPTIONS:
  -o OUTPUT_DIR       - output directory 
  -f ROOTFS_OVERLAY   - rootfs-overlay directory 
  --skip-sudo         - call docker directly, without sudo

  command line options -o, -f override config file settings.

Example:
  ./builder.sh build example.cfg -o output -f myrootfs
```

### Configuration file

The configuration file is quiet self-explanatory. The following parameters are
mandatory (prefixed with `LEDE_` for historical reasons, config works also
with OpenWrt):

* `LEDE_TARGET` - Target architecture
* `LEDE_SUBTARGET` - Sub target architecture
* `LEDE_RELEASE` - Release to use
* `LEDE_PROFILE` - Profile to use
* `LEDE_PACKAGES` - list of packages to include/exclude. Prepend package to be excluded with `-`.

`LEDE_TARGET`, `LEDE_SUBTARGET` and `LEDE_RELEASE` are used to construct the
URL of the image builder binary well as for the construction for the tag of the
docker image.

You can find the proper values (for 18.06) by browsing the OpenWrt website
[here](https://downloads.openwrt.org/releases/18.06.0/targets/)  and
[here](https://lede-project.org/toh/views/toh_admin_fw-pkg-download).

In addition the following optional parameters can be set, to further control
output and image creation:

* `OUTPUT_DIR` - path where resulting images are stored. Defaults to `output`
  in the scripts directory (can be overridden by -o parameter). Will be
  automatically created.
* `ROOTFS_OVERLAY` - path of the root file system overlay directory. Defaults
  to `rootfs-overlay` in the scripts directory (can be overridden by -f
  parameter).
* `LEDE_BUILDER_URL` - URL of the LEDE/OpenWrt image builder to use, override
   if you do not wish to use the default builder
   (`https://downloads.openwrt.org/releases/$LEDE_RELEASE/targets/$LEDE_TARGET/$LEDE_SUBTARGET/openwrt-imagebuilder-$LEDE_RELEASE-$LEDE_TARGET-$LEDE_SUBTARGET.Linux-x86_64.tar.xz`)

Use the `BASEDIR_CONFIG_FILE` variable to set locations of `OUTPUT_DIR` or
`ROOTFS_OVERLAY` relative to the configuration files location. This allows
self-contained projects outside of the lede-dockerbuilder folder. If e.g.
`ROOTFS_OVERLAY=$BASEDIR_CONFIG_FILE/rootfs-overlay` is set, then the
rootfs-overlay directory is expected to be in the same directory as the
configuration file.

[Example configuration](example-nexx-wt3020.conf) for my [NEXX
WT3020](https://wiki.openwrt.org/toh/nexx/wt3020) router, where I have an
encrypted USB disk attached so I can use it as a simple NAS with samba and ftp:

```
# profile to use: NEXX WT3020
LEDE_PROFILE=wt3020-8M
LEDE_RELEASE=18.06.2
LEDE_TARGET=ramips
LEDE_SUBTARGET=mt7620

# list packages to include in LEDE image. prepend packages to deinstall with "-".
LEDE_PACKAGES="samba36-server kmod-usb-storage kmod-scsi-core kmod-fs-ext4 ntfs-3g\
    kmod-nls-cp437 kmod-nls-iso8859-1 vsftpd cryptsetup kmod-crypto-xts\
    kmod-crypto-iv block-mount kmod-usb-ohci kmod-usb2 kmod-dm\
    kmod-crypto-misc kmod-crypto-cbc kmod-crypto-crc32c kmod-crypto-hash\
    kmod-crypto-user iwinfo tcpdump\
    -ppp -kmod-ppp -kmod-pppoe -kmod-pppox -ppp-mod-pppoe"
```

### File system overlay

Place any files and folders that should be copied to the root file system of
the resulting image to the directory pointed to by `ROOTFS_OVERLAY` (default:
`rootfs-overlay/`), which can be overridden by the -f command line option.

### Example directory structure

The following is an example directoy layout, which I use to create a customized
OpenWrt image for my [NEXX WT3020](https://wiki.openwrt.org/toh/nexx/wt3020)
router (including the generated output).

```
├── builder.sh
├── docker
│   ├── Dockerfile
│   └── etc
│        └── entrypoint.sh
├── example.cfg
├── example-openwrt.cfg
├── output
│   ├── openwrt-18.06.0-ramips-mt7620-device-wt3020-8m.manifest
│   ├── openwrt-18.06.0-ramips-mt7620-wt3020-8M-squashfs-factory.bin
│   ├── openwrt-18.06.0-ramips-mt7620-wt3020-8M-squashfs-sysupgrade.bin
│   └── sha256sums
├── README.md
└── rootfs-overlay
    ├── etc
    │   ├── config
    │   │   ├── dhcp
    │   │   ├── dropbear
    │   │   ├── firewall
    │   │   ├── network
    │   │   ├── samba
    │   │   ├── system
    │   │   ├── wireless
    │   ├── dropbear
    │   │   └── authorized_keys
    │   ├── hotplug.d
    │   │   └── block
    │   │       └── 10-mount
    │   ├── passwd
    │   ├── rc.local
    │   ├── shadow
    │   └── vsftpd.conf
    ├── README.md
    └── usr
        └── local
            └── bin
                └── fix_sta_ap.sh
```

#### Debugging

Run `./builder.sh shell CONFIGFILE` to get a shell into the docker container,
e.g. `./builder.sh shell example.cfg`.

## Examples

These examples evolved from images I use myself.

* [minimal image with LUCI web GUI for the Raspberry PI 2](example-rpi2.conf). Just ~8MB gziped. I use this image on my home dnsmasq/openvpn 'server'.  
* [image for the TP-Link WR1043ND](example-wrt1043nd.conf)
* [image with samba, vsftpd and encrypted usb disk for NEXX-WT3020](example-nexx-wt3020.conf). Is the predessor of ...
* [image with samba, vsftpd and encrypted usb disk for GINET-GL-M300N](example-ginet-gl-mt300n.conf). This is my travel router setup where I have an encrypted USB disk connected to the router.

To build an example run `./builder.sh build <config-file>`, e.g.

```shell
$ ./builder.sh build example-rpi2.conf 
```

The resulting image can be found in the `output/` directory. The [OpenWrt
wiki](https://openwrt.org/docs/guide-user/installation/generic.sysupgrade)
describes how to flash the new image in detail.

### Building a x86_64 image and running it in qemu

The [example-x86_64.conf](example-x86_64.conf) file can be used to build a
x86_64 based OpenWrt image which can also be run in qemu, e.g. if you need
a virtual router/firewall.

First build the image with `builder.sh build example-x86_64.conf`, then unpack
the resulting image with `gunzip output/openwrt-18.06.2-x86-64-combined-ext4.img.gz`.
Now the image can be started with qemu:

```
qemu-system-x86_64 \
    -enable-kvm \
    -nographic \
    -device ide-hd,drive=d0,bus=ide.0 \
    -drive file=output/openwrt-18.06.2-x86-64-combined-ext4.img,id=d0,if=none  \
    -netdev user,id=hn0 \
    -device virtio-net-pci,netdev=hn0,id=wan \
    -netdev user,id=hn1,hostfwd=tcp::5555-:22001 \
    -device virtio-net-pci,netdev=hn1,id=lan 
```

The `hostfwd=...` part can be omitted and is used in case you redirect port
22001 on your WAN adapter to port 22 of the LAN adapter, in case you want to
access SSH in the VM from your qemu-host. Check the `/etc/config/firewall` file
for details.

Qemu will assign the IP address `10.0.2.15/24` to the `WAN` interface (`eth1`)
and OpenWrt the address `192.168.1.1/24` to the `LAN` (`br-lan` bridge with
`eth0`) interface.

Note: press `CTRL-A X` to exit qemu.

## Building an OpenWrt snapshot release

To build a [snapshot](https://downloads.openwrt.org/snapshots) release, set
`LEDE_RELEASE` to `snapshots` and let `LEDE_BUILDER_URL` point to the image
builder in the snapshot dir, e.g.

```
LEDE_RELEASE=snapshots
LEDE_BUILDER_URL="https://downloads.openwrt.org/$LEDE_RELEASE/targets/$LEDE_TARGET/$LEDE_SUBTARGET/openwrt-imagebuilder-$LEDE_TARGET-$LEDE_SUBTARGET.Linux-x86_64.tar.xz" 
```

## Author

Jan Delgado

## License

Apache License 2.0
