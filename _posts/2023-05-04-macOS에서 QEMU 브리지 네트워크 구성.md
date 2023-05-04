---
layout: single
title:  "macOS에서 QEMU 브리지 네트워크 구성"
date:   2023-05-04 12:00:00 +0900
categories: 
tags: Network
---

### macOS와 TAP 디바이스

리눅스에서 QEMU로 브리지 네트워크를 구성하려면 TAP 디바이스를 이용해 가상 이더넷 브리지를 구성할 수 있다. QEMU 실행 시 `-netdev tap` 혹은 `-netdev bridge` 같은 옵션을 주면 된다.

그러나 애플의 macOS에서는 이 옵션이 먹질 않는다. 왜냐하면 맥에는 TAP 디바이스가 없기 때문이다. 맥에서 TAP 디바이스를 구현하기 위한 프로젝트인 [TunTap](https://tuntaposx.sourceforge.net/)이 있었으나 개발이 중지되어서 현재는 쓸모가 없다. 이를 대신하여 [Tunnelblick](https://tunnelblick.net/cTunTapConnections.html) VPN 소프트웨어에서 TAP 디바이스 드라이버를 제공하였으나, 이 역시 애플에서 커널 확장을 deprecate 시켜버려서 작동을 보증할 수 없는 상황이다.

### vmnet 브리지

다행히도 맥에서는 vmnet 프레임워크를 통해 가상 머신이 네트워크 패킷을 읽고 쓸 수 있도록 API를 제공한다. QEMU도 [vmnet을 지원](https://patchew.org/QEMU/20220317172839.28984-1-Vladislav.Yaroshchuk@jetbrains.com/)하므로 이 기능을 활용할 수 있다. 

`qemu-system-ARCH` 실행 시 `-nic vmnet-bridged` 옵션을 주면 된다.

예제:

```
sudo qemu-system-x86_64 \
 -machine q35 \
 -accel hvf \
 -cpu host \
 -smp 4 \
 -m 8G \
 -vga virtio \
 -usb \
 -device qemu-xhci \
 -device usb-kbd \
 -device usb-tablet \
 -nic vmnet-bridged,ifname=en7 \
 -cdrom /path/to/cdrom \
 -drive file=/path/to/disk
```

`-accel hvf`는 macOS의 Hypervisor 프레임워크를 사용한다는 의미이다. 나머지 세팅은 본인 환경에 알맞게 바꾸면 된다. 맥북의 무선랜을 사용한다면 `ifname=en0`으로 세팅하면 될 것이다.

> 테스트 환경:
>  - 하드웨어: MacBook Pro (2017, Intel 모델)
>  - 호스트: macOS 13 Ventura
>  - 게스트: Ubuntu Server 20.04 LTS
>  - QEMU 버전: 8.0

---
### 참고 자료
<https://www.sobyte.net/post/2022-10/mac-qemu-bridge>