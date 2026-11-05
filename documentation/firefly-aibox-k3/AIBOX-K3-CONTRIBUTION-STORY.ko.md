# AIBOX-K3 Armbian 기여 과정과 보드 설정 해설

발표용 정리 문서. 대상 파일: `config/boards/firefly-aibox-k3.csc`

---

## 1. 한 장 요약

| 질문 | 답 |
|---|---|
| 무엇을 기여했나 | Firefly AIBOX-K3 보드 설정 1개 파일 (이미지 지원) |
| 무엇을 하지 않았나 | SPI NOR 자동 기록, 부트로더 교체 |
| 왜 | 부트로더는 벤더 SPI NOR 펌웨어가 제공하고, Armbian은 이미지에만 관여 |
| 검증 | 실기에서 UFS 이미지 부팅 확인 |
| 영향 범위 | 보드 1개, 공용 패밀리 코드 변경 없음 |

---

## 2. 정보를 얻은 4가지 출처

기여 과정에서 근거는 항상 이 네 곳 중 하나에서 나왔습니다.

| 출처 | 무엇을 얻었나 | 예 |
|---|---|---|
| ① 빌드 프레임워크 소스 | 변수 의미, 훅 이름 규칙, 파일 타입 | `lib/functions/image/partitioning.sh`, `lib/functions/general/extensions.sh` |
| ② 같은 패밀리 보드 설정 | 관례(변수 조합, 훅 사용) | `config/boards/k3picoitx.conf`, `kickpi-k3b.conf` |
| ③ 실기 관찰 | 실제 파티션/부트 체인/DTB | `/proc/mtd`, `lsblk -f`, 부팅 로그 |
| ④ 벤더 배포물 | 공장 레이아웃, 부트 변수 | 복구 킷의 `bootfs.img`, `env_k3.txt` |

---

## 3. 보드 구조를 먼저 확인한 방법

```bash
# 저장장치와 파티션
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS

# 부트로더가 어디에 있는지
cat /proc/mtd

# 커널/DTB
uname -a
ls /boot/dtb/spacemit/ | grep aibox

# 부트 체인 로그(시리얼 115200)
#   BootROM → bootinfo → FSBL → OpenSBI → U-Boot → 커널
```

확인된 사실:

- 부트로더 체인은 **SPI NOR**(`bootinfo/fsbl/env/esos/opensbi/uboot`)에 있음
- 운영체제는 **UFS**(`armbi_boot` FAT + `armbi_root` ext4)
- 커널은 벤더 BSP `6.18.3-legacy-spacemit-k3`
- DTB는 이미 커널 패키지에 포함: `/boot/dtb/spacemit/k3-firefly-aibox-k3.dtb`

---

## 4. 설정 파일 라인별 해설과 근거

### 4.1 보드 식별

| 항목 | 값 | 의미 | 근거 |
|---|---|---|---|
| `BOARD_NAME` | `AIBOX-K3` | 표시 이름 | 제품명 |
| `BOARD_VENDOR` | `firefly` | 파일명 접두어 규칙 | `config/boards/<vendor>-<board>.<type>` |
| `BOARDFAMILY` | `spacemit-k3` | 공용 패밀리 설정 | `config/sources/families/spacemit-k3.conf` 존재 |
| `BOARD_MAINTAINER` | `leoppark94` | 유지보수자 | 기존 보드 전부 보유(`grep -L`), GitHub 핸들 |
| `INTRODUCED` | `2026` | 도입 연도 | `.csc` 237개 전부 보유 |

### 4.2 커널

| 항목 | 값 | 의미 | 근거 |
|---|---|---|---|
| `KERNEL_TARGET` | `legacy` | 벤더 BSP 6.18 | 패밀리 설정의 `case "${BRANCH}"`에서 `legacy` = `spacemit-com/linux-6.18` (vendor BSP) |
| `KERNEL_TEST_TARGET` | `legacy` | 테스트 기준 브랜치 | 같은 패밀리 보드 관례, BSP 패키징에서 사용 |

실기 확인: `uname -a` → `6.18.3-legacy-spacemit-k3`

### 4.3 부팅

| 항목 | 값 | 의미 | 근거 |
|---|---|---|---|
| `BOOT_FDT_FILE` | `spacemit/k3-firefly-aibox-k3.dtb` | 커널에 전달할 DTB | 보드 `/boot/dtb/spacemit/`에 동일 파일 존재 |

DTB가 이미 벤더 커널에 들어 있으므로 **커널 패치가 필요 없습니다.** 이것이 이 기여가 작은 이유 중 하나입니다.

### 4.4 이미지 레이아웃

| 항목 | 값 | 의미 | 근거 |
|---|---|---|---|
| `IMAGE_PARTITION_TABLE` | `gpt` | 파티션 테이블 형식 | UFS 설치/부팅 실측 |
| `OFFSET` | `12` | 이미지 앞쪽 여백(MiB) | `lib/functions/image/partitioning.sh`의 `bootstart` 계산 |
| `BOOTFS_TYPE` | `fat` | 부트 파티션 파일시스템 | 보드에서 `armbi_boot`가 vfat |
| `BOOTSIZE` | `256` | 부트 파티션 크기(MiB) | 벤더 공장 레이아웃과 동일 (`bootfs.img` = 268435456 bytes = 256MiB) |

왜 이 조합인가: 벤더가 쓰던 레이아웃과 맞춰야 **벤더 U-Boot가 파티션을 찾을 수 있습니다.**

### 4.5 부트로더를 이미지에 넣지 않는 이유

```bash
function post_config_uboot_target__firefly_aibox_k3() {
	write_uboot_platform() {
		echo "NOTE: U-Boot is stored in SPI NOR and is not written to the UFS image."
	}
}
```

| 질문 | 답 |
|---|---|
| 왜 no-op인가 | U-Boot가 UFS가 아니라 SPI NOR에 있음 |
| 근거 | `/proc/mtd`에 `uboot` 파티션 존재, 부팅 로그의 `U-Boot 2022.10_...` |
| 훅 위치 | `post_config_uboot_target`은 `lib/functions/compilation/uboot.sh`에서 호출 |
| 훅 이름 규칙 | `pre_*__<라벨>`의 라벨은 자유 형식 (`extensions.sh`가 `__` 기준으로 수집) |
| 선례 | 같은 패밀리 `k3picoitx.conf`도 동일한 no-op 패턴 사용 |

### 4.6 벤더 U-Boot용 환경 파일

```bash
function pre_umount_final_image__write_env_k3_txt() {
	local boot_dir="${MOUNT}/boot"
	cat > "${boot_dir}/env_k3.txt" <<-'EOF'
	console=ttyS0,115200
	bootdelay=5
	knl_name=Image
	dtb_dir=dtb/spacemit
	loglevel=8
	EOF
}
```

| 질문 | 답 |
|---|---|
| 무엇인가 | 벤더 부팅 변수(커널 파일명, DTB 경로, 콘솔, 부팅 지연)를 담은 파일 |
| 근거 | 공장 펌웨어의 부트 파티션에 **동일 파일이 존재** |
| 훅 | `pre_umount_final_image` (`lib/functions/image/rootfs-to-image.sh`) |
| 경로 | 다른 보드도 `${MOUNT}/boot` 사용 (`ayn-odin2.csc` 등) |
| 검증 수준 | 파일 존재 확인 + 이미지 부팅 성공. 벤더 U-Boot가 실제로 이 파일을 읽는지는 **추가 확인 항목** |

### 4.7 파일 타입: `.csc`

| 타입 | 의미 |
|---|---|
| `.conf` | 지원(supported) |
| `.csc` | 커뮤니티 지원(community supported) |
| `.wip` | 작업 중 |

`.csc`를 고른 이유: 이미지 부팅이 실기에서 검증됐고, 유지보수자가 직접 관리하기 때문입니다. 타입은 `lib/functions/main/config-prepare.sh`의 `board_types`로 결정됩니다.

---

## 5. 범위를 좁힌 이유 (NOR 제외)

기여 초기에는 SPI NOR 자동 기록까지 포함했지만, 최종적으로 **이미지 지원만** 남겼습니다.

| 이유 | 설명 |
|---|---|
| 소유권 | 부트로더 체인은 벤더 펌웨어가 관리 |
| 위험 | 파티션 단위 부분 쓰기는 실패 시 복구가 어려움 |
| 범위 | Armbian 보드 지원의 본질은 "이미지가 부팅되는가" |
| 영향 | 공용 패밀리 코드를 건드리지 않아 다른 보드 영향 0 |

결과적으로 이 파일은 **보드 설정 1개**만 추가합니다.

---

## 6. 커밋과 PR

```text
board: firefly-aibox-k3: promote to community supported

Boots UFS images with the vendor SPI NOR firmware. The image and the
installer do not write to SPI NOR.
```

PR 본문에 명시한 전제:

1. 부트로더는 벤더 NOR 그대로
2. 이미지는 UFS에 기록
3. 디스플레이/팬은 벤더 NOR 체인에 의존

---

## 7. 검증 절차 (재현)

```bash
# 1) 빌드
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy RELEASE=resolute BUILD_MINIMAL=yes

# 2) USB 기록
xzcat output/images/Armbian-*aibox-k3*.img.xz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync

# 3) 보드에서 USB 부팅 후 확인
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
findmnt -no SOURCE /
uname -a
ls /boot/dtb/spacemit/ | grep aibox
cat /proc/cmdline

# 4) UFS 설치 후 확인
#    - 스틱 제거 후 UFS 부팅
#    - findmnt / 가 UFS인지 확인
```

---

## 8. 발표 진행 순서 (제안)

| 순서 | 내용 | 시간 |
|---|---|---|
| 1 | 문제: AIBOX-K3용 Armbian 이미지가 없음 | 1분 |
| 2 | 보드 구조 파악 (NOR 부트로더 + UFS OS) | 2분 |
| 3 | 근거를 어디서 찾았나 (4가지 출처) | 2분 |
| 4 | config 라인별 해설 | 4분 |
| 5 | 범위 결정: NOR 제외 이유 | 2분 |
| 6 | 검증 결과 (UFS 부팅) | 2분 |
| 7 | Q&A | 3분 |

## 9. 예상 질문

**Q. 커널 패치는 왜 없나?**
DTB가 벤더 커널에 이미 포함되어 있어서 필요 없습니다.

**Q. NOR은 왜 지원하지 않나?**
벤더가 관리하는 영역이고, 부분 쓰기 실패 시 복구 위험이 큽니다. 이 기여는 이미지 지원에 집중했습니다.

**Q. `.wip`가 아니라 `.csc`인 이유는?**
이미지 부팅이 실기 검증됐고, 유지보수자가 관리하기로 했기 때문입니다.

**Q. `env_k3.txt`는 없어도 되지 않나?**
벤더 U-Boot가 사용하는 파일이고, 공장 펌웨어에도 동일 파일이 있습니다. 제거하면 벤더 부팅 변수를 잃을 수 있습니다.

**Q. 다른 보드에 영향이 있나?**
없습니다. 보드 파일 1개만 추가하고 공용 패밀리 설정은 변경하지 않았습니다.

**Q. 부트로더를 안 쓰면 이미지는 어떻게 부팅하나?**
벤더 NOR의 U-Boot가 UFS 파티션의 `boot.scr`을 읽어 커널을 시작합니다.

---

## 10. 참고 파일

| 목적 | 경로 |
|---|---|
| 보드 설정 | `config/boards/firefly-aibox-k3.csc` |
| 패밀리 설정 | `config/sources/families/spacemit-k3.conf` |
| 파티셔닝 로직 | `lib/functions/image/partitioning.sh` |
| 훅 메커니즘 | `lib/functions/general/extensions.sh` |
| 같은 패밀리 보드 예 | `config/boards/k3picoitx.conf` |
| 벤더 복구 킷 | `~/armbian-firefly-aibox-k3-clean/` |
