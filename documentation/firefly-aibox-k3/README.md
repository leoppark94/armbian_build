# 찾아볼꺼

> 이 메모의 질문과 답은 `AIBOX-K3-GUIDE.md`로 정리했습니다.
> 질문 → 문서 위치 매핑은 `AIBOX-K3-GUIDE.md` 0.1절에 있습니다.

## AIBOX-K3 부팅순서

```
전원 ON
|
v
BootROM (SoC 내부, 수정 불가)
|
| SPI NOR의 bootinfo를 읽음
v
bootinfo (FSBL의 위치/크기)
|
v
FSBL (U-Boot SPL, DRAM 초기화)
|
v
OpenSBI (M-mode 런타임)
|
v
U-Boot (부트로더)
|
| boot.scr 또는 env_k3.txt를 읽음
v
Linux kernel + DTB + initrd
|
v
rootfs
```
이게 정말 맞는지? 어디서 알 수 있는지?

## 파티션 확인

이거도 맞는지 확인필요. 그리고 uInitrd 이나 dtb, boot.scr 등 단어들이 뭔지 알아야함
```
SPI NOR (8MiB)                UFS (128GB)
┌─────────────────┐          ┌─────────────────┐
│ bootinfo        │          │ bootfs (256MiB) │
│ fsbl            │          │  Image          │
│ env             │          │  uInitrd        │
│ esos            │          │  dtb/           │
│ opensbi         │          │  env_k3.txt     │
│ uboot           │          │  boot.scr       │
└─────────────────┘          │ rootfs (나머지)  │
                             └─────────────────┘
```

## 칩

`팬은 CTF2301이라는 별도 칩이 제어합니다`

이걸 어떻게 아는지? 어떤 명령어로 확인할 수 있는지?

`riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: SBI MPXY extension not available` 에러메세지

## 온도

SDK는 google repo라는 처음보는 형식으로 되어있었음.

근데 armbian에서 펌웨어 태그는 `k3-br-v1.0.8` 이였고 SDK는 `k3/linux6.18_release_v1.0.0a` 태그를 사용

그래서 `FSBL/OpenSBI/ESOS/U-BOOT` 를 sdk에 있는거랑 동일하게 지정

```
/sys/class/hwmon/hwmon0/name = ctf2301
/sys/class/hwmon/hwmon0/temp1_input = 345625
```

## Armbian Uboot

어딘가 upstream uboot가 있을텐데 왜 armbian은 전용 uboot를 사용해야하는겁니까?

그리고 boot.scr 이 뭐고 env_k3.txt는 뭔가요? 뭐가다른거에요? 

write_uboot_platform_mtd 는 뭐에요?

pre_umount_final_image__firefly_aibox_k3_boot 이건 왜 여전히 필요해요?

## Armbian installer

Live USB로 어떻게 부팅하고, 어떻게 설치합니까?
좀더 자세히 알려주세요

## 15.6 BSP 온도 수정

+	if [[ ${BOARD} == "firefly-aibox-k3" ]]; then
+		# CTF2301 (hwmon0) is the fan controller, not the SoC sensor.
+		ln -fs /sys/devices/virtual/thermal/thermal_zone0/temp /etc/armbianmonitor/datasources/soctemp

이 수정이 뭔지 좀더 설명해주세요 어떻게 동작하는건가요?
