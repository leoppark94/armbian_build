# AIBOX-K3 보드 지원 개발 기록 (DEVLOG)

이 문서는 커밋 `c1b83e79d` "Add support for Firefly AIBOX-K3"와 그 이후의
수정(007 패치)까지, **무엇을 어떻게 알아냈고 왜 그런 수정을 거쳤는지** 전 과정을
기록합니다. 처음 보는 사람도 같은 과정을 따라갈 수 있게, 증거가 된 명령어와
출력을 섹션마다 실었습니다.

- 작성: 2026-10-25 실기 세션 전체 정리
- 대상 보드: Firefly AIBOX-K3 (SpacemiT K3, 16-core RISC-V, 128GB UFS + 8MiB SPI NOR)
- Armbian: `26.11.0-trunk`, `resolute`, `legacy` 계열, kernel `6.18.3-legacy-spacemit-k3`
- 펌웨어: Firefly 태그 `k3/linux6.18_release_v1.0.0a` (uboot-2022.10 + opensbi + esos)

## 0. 한눈에 보기

| 커밋 내용 | 역할 | 어떻게 도출했는지 | 섹션 |
|---|---|---|---|
| `config/boards/firefly-aibox-k3.wip` | 보드 정의(펌웨어 태그, NOR 지도, 설치 훅) | SDK/펌웨어 탐색 + 실험 | 2, 3.4 |
| `config/sources/families/spacemit-k3.conf` | esos 블롭 경로를 보드별로 덮어쓰게 | 빌드 산물의 FIT가 팬 컨트롤러를 못 잡는 문제 | 7.2 |
| u-boot 패치 001 | 부트 env 이름을 distro-boot 규약으로 | 부트 스크립트와 코드의 변수 불일치 | 7.1 |
| u-boot 패치 002 | defconfig: bootstd, 크기 제한, 로그 레벨 | SPL 크기 초과 / 부트 흐름 | 7.1 |
| u-boot 패치 003 | env 스크립트를 distro 스타일로 재작성 | 벤더 부트메뉴 구조의 한계 | 7.1 |
| u-boot 패치 004 | miiphy_init()를 board_init에서 일찍 | bootefi 접근장애 재현 | 7.1 |
| u-boot 패치 005 | 벤더 PRINT_TIMESTAMP 버그 되돌리기 | printf가 글자를 깎아먹는 현상 | 7.1 |
| u-boot 패ッチ 006 | tools/Makefile 감싸기 규칙 수정 | 빌드 순환 의존 | 7.1 |
| `esos.itb` (Firefly 빌드) | ESOS(임베디드 시스템 OS) 펌웨어 | SDK의 Firefly 빌드와 동일 링크 사용 | 7.2 |
| hardware-monitor soctemp | CTF2301 = 팬 컨트롤러, SoC 온도는 thermal_zone0 | hwmon0의 name 확인 | 7.3 |
| **(신규) u-boot 패치 007** | USB 호스트 포트 VBUS 전원 | PORTSC + GPIO 바이센트 실험 | 5 |
| **(신규) documentation/** | 이 문서, 이전 문서, 도구 | 위 전체 | 8 |

실기 검증 결과:

| 목표 | 결과 |
|---|---|
| USB 스틱 라이브 부팅 (U-Boot 경유) | 성공 — `1 Storage Device(s) found` → `Booting Armbian (usb 0:1)` |
| 수동 조작 없는 콜드부팅에서 동일 | 성공 — 007(FIT 레벨 검증) 적용 후 자동 열거 |
| `armbian-install`로 UFS에 설치 | 성공 — `--boot spi` 모드, 4 KiB 섹터 대상 정상 파티셔닝 |
| 설치된 시스템이 NOR U-Boot에서 부팅 | 성공 — 재부팅 후 SSH/login 정상 |

---

## 1. 증상과 접근 환경

**증상**: 빌드 출력물을 USB 스틱에 구우면 U-Boot가 스틱을 전혀 인식하지 못한다.
라이브 USB 부팅도, `armbian-install` 시험도 불가능했다.

**접근 환경** (케이블이 1개라 콘솔과 fastboot를 동시에 못 쓰는 제약):

| 경로 | 방법 |
|---|---|
| AIBOX SSH | `leopark@192.168.1.<DHCP>` (세션 예: .12, .84, .181), 암호 `fj4923` |
| 콘솔 | 192.168.1.10 호스트의 CDC-ACM(`/dev/ttyACM0`) 115200 |
| fastboot | 전면 Type-C + Boot 버튼 (SPL fastboot) |

콘솔 데몬은 재열거마다 보레이트가 9600으로 리셋되어 가비지를 뱉었다.
`tools/console_daemon2.py`는 열 때마다 115200 raw를 강제하고 재열거에 재접속한다.

---

## 2. 지형 파악: 무엇이 어디에 붙어 있는가

수정을 하기 전에 보드의 배선과 부트체인을 먼저 고정했다. 모든 "포트 어느 것이
어느 컨트롤러" 결론은 Linux에 옮겨 심고 sysfs로 확인했다.

### 2.1 부트체인과 NOR 지도

```text
전원 ON → BootROM → bootinfo → FSBL(SPL, DRAM) → OpenSBI → U-Boot → boot.scr/env → Linux
```

SPI NOR (8MiB)은 kernel cmdline의 `mtdparts=`와 동일하게 6개 영역:

```text
mtd0 bootinfo 128K | mtd1 fsbl 512K | mtd2 env 64K | mtd3 esos 1M
| mtd4 opensbi 384K | mtd5 uboot -(나머지, 약 5.9M)
```

U-Boot FIT(`u-boot.itb`)에는 `uboot`(커널 아님, U-Boot 본체)과 `fdt_1..fdt_14`
보드별 DTB가 들어 있다. AIBOX-K3 = `fdt_14` (`k3-firefly-aibox-k3.dtb`).

### 2.2 포트 ↔ 컨트롤러 지도 (실측)

```text
$ readlink -f /sys/bus/usb/devices/usb5 /sys/bus/usb/devices/usb7 /sys/bus/usb/devices/usb1
.../soc/81700000.usb3/xhci-hcd.2.auto/usb5   ← 스틱이 꽂힌 후면 USB-A
.../soc/81a00000.usb3/xhci-hcd.3.auto/usb7   ← r8152 이더넷 동글 포트
.../soc/c0a00000.usb2/xhci-hcd.0.auto/usb1   ← 온보드 4포트 USB2 허브
```

| 물리 | 컨트롤러 | Linux 버스 | U-Boot DTB 상태 |
|---|---|---|---|
| 전면 Type-C (USB3.0 DRD) | cad00000 (usb3@0) | — | okay / dr_mode=peripheral (콘솔·fastboot용) |
| 전면 Console | 보드가 CDC-ACM 기기로 열거 | 콘솔 호스트 | — |
| **후면 USB-A (테스트 스틱)** | **81700000 (usb3@2)** | 5-1 (5000M) | **disabled ← 문제 ②** |
| 후면 USB-A (동글) | 81a00000 (usb3@3) | 7-1 | disabled |
| 온보드 허브 | c0a00000 (usb2@0) | 1-1 (4포트) | disabled |

이 지도가 그 이후 모든 판단의 기준이 됐다. "뒷면 포트 = 81400000"이라는
초기 추측은 이 측정으로 정정됐고, 문서에 정정 기록을 남겼다.

---

## 3. 진단 ①: 빌드 이미지가 4 KiB 논리 섹터 GPT

### 3.1 발견

보드 파일(`config/boards/firefly-aibox-k3.wip`):

```bash
IMAGE_PARTITION_TABLE="gpt"
SECTOR_SIZE="4096"
OFFSET="12"
```

`lib/functions/image/partitioning.sh`는 이 값을 `sfdisk --sector-size`와
`losetup -b`로 넘긴다. 그래서 `.img`의 GPT 헤더가 byte 512(LBA1)가 아니라
**byte 4096(LBA8)** 에 있다.

### 3.2 왜 USB 스틱에서 안 읽히는가

- 일반 스틱의 `logical_block_size` = 512. GPT의 표준 위치 LBA1 = byte 512.
  스틱의 도구들은 byte 512를 읽으니 테이블을 못 찾는다: `unrecognised disk label`
- U-Boot는 `usb_stor_get_info()`의 SCSI READ CAPACITY 응답으로 블록 크기를 정하고
  `part_get_info()`가 `blksz × 1` 위치의 헤더를 읽는다. 4 KiB 이미지는 UFS(4 KiB)에는
  맞지만 스틱(512)에는 안 맞는다.
- ext4도 4 KiB 블록이라 512 섹터 장치에 못 마운트한다:
  `EXT4-fs (sdb2): bad geometry: block size 4096 > logical sector size 512`

### 3.3 조치 — 512 섹터로 변환

`tools/mk512.py`: GPT를 재작성(파티션 GUID 유지). `tools/make-512-live-usb.sh`:
파일시스템을 재생성하고 내용을 복사한다. 변환 후:

| 항목 | 원본 | 변환본 |
|---|---|---|
| GPT 헤더 | byte 4096 | byte 512 |
| bootfs | FAT16, 4096 B 섹터 | FAT16, 512 B 섹터 |
| rootfs | ext4, 4096 B 블록 | ext4, 1024 B 블록 |
| PARTUUID/라벨 | 그대로 | 그대로 |

### 3.4 변환 도구의 버그: FAT 볼륨 시리얼

변환 후 라이브 부팅이 `/boot` 마운트에서 죽었다(e emergency mode):

```text
# blkid (스틱)     /dev/sdb1 ... UUID="3EA2-3776"      ← mkfs.vfat가 랜덤 생성
# /etc/fstab       UUID=4A74-C181 /boot vfat           ← 원본 시리얼 참조
```

원인: PARTUUID와 ext4 UUID는 보존했는데 **FAT 볼륨 시리얼(-i)은 빠뜨렸다.**
`make-512-live-usb.sh`를 고쳐 소스의 시리얼을 자동 감지해 보존한다.
수정 후 동일 이미지에서 emergency 없이 `/boot`가 마운트됐다.

```bash
run mkfs.vfat -F 16 -i "$BOOT_SERIAL" -n armbi_boot "${LOOP}p1"
```

교훈: 라이브 부팅의 실패는 반드시 한 개만이 아니었다. **fstab이 참조하는
모든 식별자는 변환 시 "전부" 나열하고 하나씩 검증하자.**

---

## 4. 진단 ②: 배포 U-Boot DTB가 스틱 포트(usb3@2)를 disabled로 둔다

### 4.1 발견

소스 DTS(`arch/riscv/dts/k3-firefly-aibox-k3.dtsi`)는 `usb3_portc`(=usb3@2)를
okay로 선언하지만, **배포된 `u-boot.itb`의 `fdt_14`는 disabled로 만들어져
나왔다.** 소스 빌드와 펌웨어 산물이 어긋난 상태.

| 글루 노드 | 배포 fdt_14 | 소스 |
|---|---|---|
| usb3@0 (cad00000) | okay, dr_mode=peripheral | okay |
| usb3@1 (81400000) | okay, dr_mode=host | okay |
| **usb3@2 (81700000)** | **disabled** | okay |
| usb3@3 (81a00000) | disabled | disabled |

### 4.2 조치 — FIT의 DTB를 직접 고쳐 NOR에 기록

`tools/patch-fit.py`: FIT의 `fdt_14`에서 `/soc/usb3@2`, `/soc/phy@81800000`,
`/soc/phy@82000000`의 status를 okay로 바꾸고, 그 외 모든 속성을 동일 유지한다
(430노드 비교로 차이 3개 확인). FIT의 crc32 해시도 재계산한다.

NOR 기록은 라이브 시스템에서 `flashcp`로 한다(SPL은 건드리지 않는다):

```bash
dd if=/dev/mtd6 of=mtd6-backup.bin        # 항상 백업 먼저
flashcp -v u-boot-patched.itb /dev/mtd6
cmp -n <size> /dev/mtd6 u-boot-patched.itb && echo OK
reboot
```

기록 후 `dm tree`로 검증:

```text
phy 5 [ + ] k3-usb3-phy       phy@82000000
usb  1 [ + ] dwc3-generic-host  dwc3@81700000
```

컨트롤러와 PHY가 모두 프로브 성공. 그러나 **여전히 장치가 열거되지 않았다** —
원인 ③으로 이어진다.

---

## 5. 진단 ③: VBUS 5V가 켜지지 않는다 (마지막 원인)

### 5.1 우선 레지스터를 읽었다

U-Boot 프롬프트(`s` 키로 autoboot 정지)에서 xHCI PORTSC를 직접 읽었다.
DWC3의 xHCI 캡 워드(0x01100020)의 HCLENGTH=0x20, PORTSC = base + caplen + 0x400.

```text
=> md.l 0x81700420 2        (스틱 포트)
81700420: 000002a0          ; CCS=0, PLS=5(Polling), PP=1
=> md.l 0x81400420 2        (빈 포트)
81400420: 000002a0          ; 완전히 동일
```

**PLS=Polling(수신 감지 반복)을 유지하는데 CCS=0 = 전기적으로 아무것도 안 보인다.**
`usb reset`을 아무리 반복해도 같았다. 타이밍/스캔 소프트웨어가 아니다.
APMU 매트릭스(`0xd42829d8` = 0xf6: bit1 USB-C 모드, bit2 USB-B, bit4 X8 disable)도
정상이었다 — "USB가 안 되는" 대부분의 후보를 이 단계에서 제거했다.

### 5.2 커널 DTB에서 전원 스위치를 발견

커널 `/boot/dtb/spacemit/k3-firefly-aibox-k3.dtb`에는 USB 5V GPIO 스위치 5개가
`regulator-fixed` + `regulator-always-on` + `regulator-boot-on`으로 있다.
**U-Boot DTB에는 이런 노드가 하나도 없다.**

| 커널 레일 | GPIO (bank.pin = SoC) | 대상 |
|---|---|---|
| vcc5v0_usb30_host1 | (2,14) = **GPIO_78** | 후면 USB-A 스틱 포트 |
| vcc5v0_usb30_host0 | host1 레일을 vin으로 공유 | |
| vcc5v0_usb30_host | (2,13) = GPIO_77 | 다른 뒷면 레일 |
| vcc5v0_hub20 | (0,2) = GPIO_2 | 온보드 허브 |
| usb30_phy_5v | (1,21) = **GPIO_53** | USB3 PHY 아날로그 5V |

보드 dtsi가 `GPIO_53`을 gpio-range로 특별히 지정한 이유가 바로 이 핀이었다.

### 5.3 GPIO 바이센트 실험 (필요 조건을 좁혔다)

U-Boot 프롬프트에서 `gpio`/`usb reset`을 한 조건씩 조합했다.

| 상태 | 결과 |
|---|---|
| GPIO 없이 (베이스라인) | 0 Storage Device(s) |
| 78 단독 → 즉시 rescan | 0 (VBUS 인가 후 스틱 준비 시간 필요) |
| 78 → 8초 대기 후 rescan | **1 Storage Device(s) — 성공** |
| {78,53} | 1 |
| {77} 단독 | 0 |
| {77,2} | 0 |
| {78,77,2,53} | 1 |
| {77,78,53} → {78,53} | 1 |

결론: **필요한 것은 GPIO_78 하나** (+ 포트 파워 후 스틱 준비 대기).
GPIO_53(phy 5V)은 필요하지 않았고, 이전 "78 단독 실패"는 대기 시간이었다.

### 5.4 드라이버가 이미 준비되어 있었다

`drivers/usb/dwc3/dwc3-generic.c`의 `dwc3_generic_host_probe()`:

```c
#if CONFIG_IS_ENABLED(K1_X_BOARD_ASIC) || CONFIG_IS_ENABLED(K3_BOARD_ASIC)
	if (device_is_compatible(dev->parent, "spacemit,k3-dwc3")) {
		rc = device_get_supply_regulator(dev->parent, "vbus-supply",
						&priv->vbus_supply);
		...
		if (priv->vbus_supply)
			regulator_set_enable(priv->vbus_supply, true);
	}
#endif
```

BSP defconfig에 `CONFIG_K3_BOARD_ASIC=y`, `CONFIG_DM_REGULATOR_FIXED=y`가 이미
있다. **부족한 것은 DTS의 레귤레이터 노드와 vbus-supply 참조 하나뿐.**

### 5.5 패치 007

`patch/u-boot/legacy/u-boot-spacemit-k3/board_firefly-aibox-k3/007-Firefly-AIBOX-K3-add-USB-vbus-supply-for-host-ports.patch`:

보드 전용 패치는 패치 큐의 `board_${BOARD}` 하위 디렉토리에 둔다.
`advanced_patch`는 `board_<BOARD>`를 기본 큐보다 먼저 조사하고 같은 이름은
앞쪽이 이긴다. 이 보드는 Firefly 소스를 쓰므로 001-004의 재베이스 버전과 007이
`board_firefly-aibox-k3/`에 있고, 다른 spacemit-k3 보드는 계속 가족 기본 큐를
쓴다. 005/006은 두 소스에서 동일하므로 기본 큐에만 둔다.

1. 루트에 `vcc5v0_usb30_host1` (`regulator-fixed`, gpio = `<&gpio 78 0>`,
   enable-active-high, always-on, boot-on)
2. `&usb3_portc { vbus-supply = <&vcc5v0_usb30_host1>; };`

소스 빌드 전 검증으로, 같은 내용을 FIT의 `fdt_14`에 넣는 도구
(`tools/fit-vbus-surgery.py`)를 만들어 NOR에 기록했다.

---

## 6. 종단 검증 (실기)

### 6.1 패치 전: 프롬프트에서 GPIO 수동 인가

```text
=> gpio set 78
=> usb reset
scanning usb for storage devices... 1 Storage Device(s) found
=> boot
Loading Armbian environment (usb 0:1)
Loading spacemit/k3-firefly-aibox-k3.dtb (usb 0:1)
26940733 bytes read in 272 ms (94.5 MiB/s)     ← 스틱의 FAT에서 커널
Booting Armbian (usb 0:1)
```

### 6.2 패치 후: 콜드부팅, 수동 조작 전혀 없음

`u-boot-vbus.itb`를 NOR에 기록하고 전원만 켰다:

```text
scanning bus dwc3@81400000 for devices... 1 USB Device(s) found
scanning bus dwc3@81700000 for devices... 2 USB Device(s) found
       scanning usb for storage devices... 1 Storage Device(s) found
Loading Armbian environment (usb 0:1)
Booting Armbian (usb 0:1)
```

라이브 시스템 부팅 → `mount`: `/dev/sdb1 on /boot type vfat` (시리얼 보존 덕에
fstab 수정 없이 마운트) → SSH 정상. **첫 전원에서 스틱을 스스로 찾아 부팅한다.**

### 6.3 armbian-install (라이브 시스템에서 UFS로 설치)

```bash
sudo armbian-install --target /dev/sda --boot spi --fs ext4 --yes
# scenario: spi install to /dev/sda completed
```

- 4 KiB 섹터 UFS에 GPT/ext4를 정상 기록 (1 MiB 정렬, 단일 ext4 파티션, legacy_boot)
- NOR의 U-Boot는 그대로 두고 부트 파일만 새로 쓰는 `spi` 모드가 이 보드 구조에 맞음
- 재부팅: NOR U-Boot → `Loading Armbian environment (scsi 0:1)` → 설치 시스템 부팅
  (4 KiB 대상에서 `load scsi 0:1`이 ext4를 그대로 읽는 것도 확인)

불발 케이스 (문서로 남김):

```text
sudo armbian-install --target /dev/sda --boot ufs --fs ext4 --yes
# scenario: no bootloader method for 'ufs' on this system
#   (u-boot hooks or grub-install missing) - refusing to modify /dev/sda
```

원인: SpacemiT 가족 hook에 `write_uboot_platform_ufs`가 없다.
이 보드는 U-Boot가 NOR에 상주하므로 `spi` 모드가 맞는 구조이다.
(보드 안티 낙하 방지: 설치기가 "지울 수 있는" 디스크를 미리 검증하는 동작은
올바른 설계이다.)

---

## 7. 커밋 c1b83e79의 다른 수정들 (도출 과정)

### 7.1 U-Boot 패치 (001-006)

| # | 내용 | 왜 |
|---|---|---|
| 001 | `boot_device`→`devtype`, `boot_devnum`→`devnum`, `bootfs_part`→`distro_bootpart` | 벤더 C 코드가 세팅하는 env 이름이 배포 bootfunc/boot.scr과 어긋나 USB/기기 순서가 무시됨 |
| 002 | defconfig: `CONFIG_BOOTSTD_FULL`, `CONFIG_BOOTCOMMAND="run distro_bootcmd"`, `BOOTDELAY`, `LOGLEVEL=1`, SPL 크기 한계 유지(0x72F00 < BootROM 0x74000) | 벤더 부트메뉴 대신 distro 부트 사용; SPL은 BootROM 다운로드 크기를 넘으면 부팅 불가 |
| 003 | env 스크립트 재작성: `bootfunc` (usb→sd→nvme→scsi→mmc) + extlinux/boot.scr/EFI 탐색 | extlinux.conf 등 표준 부트 아티팩트를 찾는 distro 흐름이 Armbian 이미지와 맞음 |
| 004 | `board_init()`에서 `miiphy_init()` 일찍 호출 | bootefi가 네트워크 스택에 접근하면 접근장애로 크래시 (dhcp/ping 없이 부팅할 때) |
| 005 | 벤더 `CONFIG_PRINT_TIMESTAMP` 패치 되돌리기 | vprintf가 버퍼 오프셋을 잘못 다루어 출력이 깨짐 |
| 006 | tools/Makefile의 `%.c` 랩 규칙 분리 | make 버전에 따라 패턴 규칙의 순환 의존이 발생 |

패치 헤더: 001-006의 상당수는 Patrick Yavitz 등에서 rebase한 것임을
`Rebased onto k3-br-v1.0.x` 주석으로 기록했다. 태그는 SDK와 동일한
`k3/linux6.18_release_v1.0.0a`로 고정했다(Firefly FSBL/OpenSBI/ESOS/U-Boot 조합).

### 7.2 ESOS 블롭과 그 경로

ESOS(임베디드 시스템 OS)는 Firefly가 빌드한 `esos.itb`만으로 교체했다.
`packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb`가 공용 디스크가 아니라
보드 전용일 수 있게, 가족 설정의 `uboot_custom_postprocess()`에
`K3_ESOS_BLOB` 재정의 지점을 두었다:

```bash
local esos_blob="${K3_ESOS_BLOB:-${SRC}/packages/blobs/riscv64/spacemit-k3/esos.itb}"
```

(도출: 빌드 산물의 FIT가 팬 컨트롤러 CTF2301을 못 잡아 `eSPI not ready`와 함께
온도 센서가 사라졌다 — SDK의 Firefly 빌드의 esos로 교체해 해결; 다른 보드가
벤더 기본 esos를 계속 쓰도록 기본값을 유지.)

### 7.3 hardware-monitor soctemp

```bash
if [[ ${BOARD} == "firefly-aibox-k3" ]]; then
	# CTF2301 (hwmon0) is the fan controller, not the SoC sensor.
	ln -fs /sys/devices/virtual/thermal/thermal_zone0/temp /etc/armbianmonitor/datasources/soctemp
```

도출: `/sys/class/hwmon/hwmon0/name` = `ctf2301` — hwmon0은 SoC 온도계가 아니라
**팬 컨트롤러 칩**이었다. SoC 온도는 `thermal_zone0`에 있다.
(armbian-hardware-monitor가 hwmon0/temp1_input을 SoC 온도로 연결하는 관례를
이 보드에서만 예외 처리.)

### 7.4 설치·관련 훅 (wip)

- `K3_MTDPARTS`: SPI NOR 6영역 지도. kernel cmdline의 mtdparts와 일치. Linux/`armbian-install`/U-Boot가 같은 이름(fsbl/env/.../uboot)으로 말하게 함.
- `write_uboot_platform_mtd()`: NOR 파티션은 `mtdpart` 이름으로만 찾는다.
  `/proc/mtd`에서 이름 → `/dev/mtdX`를 풀어 flashcp로 기록.
- `pre_install_kernel_debs__extra_boot_args`: `extraargs`에 `net.ifnames=0` +
  `mtdparts=` 추가 — 이더넷 이름을 안정화하고 NOR 영역이 커널에도 보이게.
- `pre_umount_final_image..._boot`: 이미지의 bootfs에 Firefly의 DTB 구조
  (`spacemit/k3-firefly-aibox-k3.dtb`)를 정리.

---

## 8. 문서와 도구 인벤토리

documentation/firefly-aibox-k3/ (저장소에 넣는 것):

| 파일 | 내용 |
|---|---|
| `AIBOX-K3-SUPPORT-DEVLOG.md` (이 문서) | 커밋 전 과정 + 진단 기록 |
| `LIVE-USB-INSTALL.md` | 라이브 USB 만들기/설치 절차 |
| `tools/` | 아래 도구 7개 |

개인 학습 노트(`AIBOX-K3-GUIDE.md`, `ARMBIAN-AIBOX-K3-SPI-NOR.ko.md`, `README.md`
의 질문 목록)는 로컬에만 둔다. 저장소에는 넣지 않는다.

| 파일 | 용도 |
|---|---|
| `mk512.py` | 4 KiB 섹터 GPT → 512 섹터 GPT 재작성 |
| `make-512-live-usb.sh` | 파일시스템 재생성(FAT 시리얼 보존) + 복사 + 검증 |
| `fix-backup-gpt.py` | 스틱 > 이미지일 때 백업 GPT 정리 |
| `patch-fit.py` | FIT의 fdt_14 속성 편집 + 해시 재계산 |
| `fit-vbus-surgery.py` | fdt_14에 VBUS 레귤레이터+vbus-supply 추가 (007의 FIT 레벨 검증) |
| `stop-autoboot.sh` | autoboot 카운트다운 감지 후 's' 연사 |
| `console_daemon2.py` | 콘솔 데몬(115200 강제, 재열거 재접속) |

---

## 9. 남은 upstream 기여 항목

1. **패치 007을 소스 빌드로 확정** — 현재까지는 FIT 레벨 검증. 다음 빌드 사이클에
   007을 받아 다시 빌드하면 NOR 대신 자연스럽게 통합됨.
2. **armbian-config(module_partitioner)**: `--boot ufs`에 SpacemiT 가족용
   `write_uboot_platform_ufs` 훅이 없다. NOR에 상주하는 이 보드 계열에서는
   `spi` 모드가 정답이므로, 메뉴에서 ufs 모드를 숨기거나 훅을 추가하는 방향.
3. **SECTOR_SIZE 문서화**: UFS(4 KiB) 타깃에는 필수지만 USB 스틱에는 변환이 필요.
   `LIVE-USB-INSTALL.md`의 512 섹터 이미지 절차가 정식 절차.
4. **전면 Type-C 라이브 부팅은 범위 밖** (결정) — 콘솔/fastboot용으로 유지.
5. 두 번째 후면 포트(usb3_portb, GPIO_77) VBUS는 기기 없이 검증 불가 —
   기기로 확인하는 즉시 007과 동일한 방식으로 추가.