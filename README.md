English | [简体中文](README.zh-cn.md)

# Table of contents
- [Overview](#overview)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Native Bridge Support](#native-bridge-support)
- [GMS Support](#gms-support)
- [WebRTC Streaming](#webrtc-streaming)
- [How To Build](#how-to-build)
- [Troubleshooting](#troubleshooting)
- [Note](#note)
- [Contact Me](#contact-me)
- [License](#license)

## Overview
*ReDroid* (*Re*mote an*Droid*) is a GPU accelerated AIC (Android In Cloud) solution. You can boot many
instances in Linux host (`Docker`, `podman`, `k8s` etc.). *redroid* supports both `arm64` and `amd64` architectures. 
*ReDroid* is suitable for Cloud Gaming, Virtualise Phones, Automation Test and more.

![Screenshot of ReDroid 11](./assets/redroid11.png)

Currently supported:
- Android 13 (`redroid/redroid:13.0.0-latest`, `redroid/redroid:13.0.0-amd64`, `redroid/redroid:13.0.0-arm64`)
- Android 12 (`redroid/redroid:12.0.0-latest`, `redroid/redroid:12.0.0-amd64`, `redroid/redroid:12.0.0-arm64`)
- Android 12 64bit only (`redroid/redroid:12.0.0_64only-latest`, `redroid/redroid:12.0.0_64only-amd64`, `redroid/redroid:12.0.0_64only-arm64`)
- Android 11 (`redroid/redroid:11.0.0-latest`, `redroid/redroid:11.0.0-amd64`, `redroid/redroid:11.0.0-arm64`)
- Android 10 (`redroid/redroid:10.0.0-latest`, `redroid/redroid:10.0.0-amd64`, `redroid/redroid:10.0.0-arm64`)
- Android 9 (`redroid/redroid:9.0.0-latest`, `redroid/redroid:9.0.0-amd64`, `redroid/redroid:9.0.0-arm64`)
- Android 8.1 (`redroid/redroid:8.1.0-latest`, `redroid/redroid:8.1.0-amd64`, `redroid/redroid:8.1.0-arm64`)


## Getting Started
*redroid* should capabale running on any linux (with some kernel features enabled).

Quick start on *Ubuntu 20.04* here; Check [deploy section](deploy/README.md) for other distros.

```bash
## install docker https://docs.docker.com/engine/install/#server

## install required kernel modules
apt install linux-modules-extra-`uname -r`
modprobe binder_linux devices="binder,hwbinder,vndbinder"
modprobe ashmem_linux


## running redroid
docker run -itd --rm --privileged \
    --pull always \
    -v ~/data:/data \
    -p 5555:5555 \
    redroid/redroid:11.0.0-latest

### Explanation:
###   --pull always    -- use latest image
###   -v ~/data:/data  -- mount data partition
###   -p 5555:5555     -- expose adb port


## install adb https://developer.android.com/studio#downloads
adb connect localhost:5555
### NOTE: change localhost to IP if running redroid remotely

## view redroid screen
## install scrcpy https://github.com/Genymobile/scrcpy/blob/master/README.md#get-the-app
scrcpy -s localhost:5555
### NOTE: change localhost to IP if running redroid remotely
###     typically running scrcpy on your local PC
```

## Configuration

```
## running redroid with custom settings (custom display for example)
docker run -itd --rm --privileged \
    --pull always \
    -v ~/data:/data \
    -p 5555:5555 \
    redroid/redroid:11.0.0-latest \
    androidboot.redroid_width=1080 \
    androidboot.redroid_height=1920 \
    androidboot.redroid_dpi=480 \
```

| Param | Description | Default |
| --- | --- | --- |
| `qemu` | export param with the "ro.kernel." prefix<br>**NOT** `QEMU-KVM` related, and plan to remove | 1 |
| `androidboot.hardware` | specify `ro.boot.hardware` prop | redroid |
| `androidboot.redroid_width` | display width | 720 |
| `androidboot.redroid_height` | display height | 1280 |
| `androidboot.redroid_fps` | display FPS | 30(GPU enabled)<br> 15 (GPU not enabled)|
| `androidboot.redroid_dpi` | display DPI | 320 |
| `androidboot.use_memfd` | use `memfd` to replace deprecated `ashmem`<br>plan to enable by default | false |
| `androidboot.use_redroid_overlayfs` | use `overlayfs` to share `data` partition<br>`/data-base`: shared `data` partition<br>`/data-diff`: private data | 0 |
| `androidboot.redroid_net_ndns` | number of DNS server, `8.8.8.8` will be used if no DNS server specified | 0 |
| `androidboot.redroid_net_dns<1..N>` | DNS | |
| `androidboot.redroid_net_proxy_type` | Proxy type; choose from: `static`, `pac`, `none`, `unassigned` | |
| `androidboot.redroid_net_proxy_host` | | |
| `androidboot.redroid_net_proxy_port` | | 3128 |
| `androidboot.redroid_net_proxy_exclude_list` | comma seperated list | |
| `androidboot.redroid_net_proxy_pac` | | |
| `androidboot.redroid_gpu_mode` | choose from: `auto`, `host`, `guest`;<br>`guest`: use software rendering;<br>`host`: use GPU accelerated rendering;<br>`auto`: auto detect | `auto` |
| `androidboot.redroid_gpu_node` | | auto-detect |
| `ro.xxx`| **DEBUG** purpose, allow override `ro.xxx` prop; For example, set `ro.secure=0`, then root adb shell provided by default | |


## Native Bridge Support
It's possible to run `arm` Apps in `x86` *ReDroid* instance via `libhoudini`, `libndk_translator` or `QEMU translator`.

Take `libndk_translator` as an example:

``` bash
# grab libndk_translator libs from Android 11 Emulator
find /system \( -name 'libndk_translation*' -o -name '*arm*' -o -name 'ndk_translation*' \) | tar -cf native-bridge.tar -T -

# example structure, be careful the file owner and mode

system/
├── bin
│   ├── arm
│   └── arm64
├── etc
│   ├── binfmt_misc
│   └── init
├── lib
│   ├── arm
│   └── libnb.so
└── lib64
    ├── arm64
    └── libnb.so
```

```dockerfile
# Dockerfile
FROM redroid/redroid:11.0.0-amd64

ADD native-bridge.tar /
```

```bash
# build docker image
docker build . -t redroid:11.0.0-amd64-nb

# running
docker run -itd --rm --privileged \
    -v ~/data11-nb:/data \
    -p 5555:5555 \
    redroid:11.0.0-amd64-nb \
    ro.product.cpu.abilist=x86_64,arm64-v8a,x86,armeabi-v7a,armeabi \
    ro.product.cpu.abilist64=x86_64,arm64-v8a \
    ro.product.cpu.abilist32=x86,armeabi-v7a,armeabi \
    ro.dalvik.vm.isa.arm=x86 \
    ro.dalvik.vm.isa.arm64=x86_64 \
    ro.enable.native.bridge.exec=1 \
    ro.dalvik.vm.native.bridge=libndk_translation.so \
    ro.ndk_translation.version=0.2.2
```

Take a look at https://gitlab.com/android-generic/android_vendor_google_emu-x86 to extract automatically libndk_translator from the Android 11 emulator images. 

After following the guide on "Building" section, you will get native-bridge.tar under vendor/google/emu-x86/proprietary.

If you find errors in using libndk_translator, please try the following:

- YOU MUST HAVE binfmt_misc kernel module loaded for supporting other binaries formats! If you have not loaded it already:

  ```bash
  sudo modprobe binfmt_misc
  ```

  or add binfmt_misc to /etc/modules to autoload it at boot (for example in Ubuntu). 

  Check your specific distribution wiki/docs if you don't have binfmt_misc module and you want to install it, or how to autoload the module at boot.

- Extract the native bridge archive, preserving the permissions, set specific permissions for allowing init file to be executed and traverse of important dirs:

  ```bash
  mkdir native-bridge
  cd native-bridge
  sudo tar -xpf ../native-bridge.tar `#or path to your actual native bridge tarball`
  sudo chmod 0644 system/etc/init/ndk_translation_arm64.rc
  sudo chmod 0755 system/bin/arm
  sudo chmod 0755 system/bin/arm64
  sudo chmod 0755 system/lib/arm
  sudo chmod 0755 system/lib64/arm64
  sudo chmod 0644 system/etc/binfmt_misc/*
  sudo tar -cpf native-bridge.tar system
  ```

  Move or copy your new native-bridge.tar into the dir where you have written your Dockerfile, and rebuild again the new image with native bridge support. 
  
  You must use sudo or a root shell to preserve the permissions and owners of the files.

## GMS Support

It's possible to add GMS (Google Mobile Service) support in *ReDroid* via [Open GApps](https://opengapps.org/), [MicroG](https://microg.org/) or [MindTheGapps](https://gitlab.com/MindTheGapps/vendor_gapps).

Check [android-builder-docker](./android-builder-docker) for details.


## WebRTC Streaming
**CALL FOR HELP**

Plan to port `WebRTC` solutions from `cuttlefish`, including frontend (HTML5), backend and many virtual HALs.

## How To Build
It's Same as AOSP building process. But I suggest to use `docker` to build.

Check [android-builder-docker](./android-builder-docker) for details.

## Troubleshooting
- Container disappeared immediately
> make sure the required kernel modules are installed; run `dmesg -T` for detailed logs

- Container running, but adb cannot connect (device offline etc.)
> run `docker exec -it <container> sh`, then check `ps -A` and `logcat`
>
> try `dmesg -T` if cannot get a container shell


## Note
- `redroid` require `pid_max` less than 65535, or else may run into problems. Change in host OS, or add `pid_max` separation support in PID namespace
- SElinux is disabled in *ReDroid*;
- CGroups errors ignored; some (`stune` for example) not supported in generic linux.
- `procfs` not fully seperated with host OS; Community use `lxcfs` and some cloud vendor ([TencentOS](https://github.com/Tencent/TencentOS-kernel)) enhanced in their own kernel.
- `vintf` verify disabled

## Contact Me
- ziyang.zhou@outlook.com
- remote-android.slack.com (invite link: https://join.slack.com/t/remote-android/shared_invite/zt-q40byk2o-YHUgWXmNIUC1nweQj0L9gA)

## License
*ReDroid* itself is under [Apache License](https://www.apache.org/licenses/LICENSE-2.0), since *ReDroid* includes 
many 3rd party modules, you may need to examine license carefully.

*ReDroid* kernel modules are under [GPL v2](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
