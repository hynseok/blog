---
title: "vmlinux 란?"
date: 2025-09-29T07:00:00+09:00
# weight: 1
# aliases: ["/first"]
tags: ["linux", "kernel"]
author: "Me"
categories: ["Kernel"]
# author: ["Me", "You"] # multiple authors

showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "linux의 vmlinux와 vmlinuz가 무엇인지 알아보는 글"
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "posts/cover/linux.png" # image path/url
    alt: "cover-image"

editPost:
    URL: "https://github.com/hynseok/blog/blob/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

# vmlinux 란?

리눅스 커널을 한 번이라도 빌드한 경험이 있다면 `vmlinux`와 `vmlinuz`라는 파일을 본 경험이 있을 것입니다. 빌드 경험이 없더라도, 리눅스 루트 파일시스템을 돌아다니다가 `/boot` 디렉토리 내에 있는 `vmlinuz` 혹은 `bzImage`를 보았을 수도 있고요.

이번 글에서는 위 파일들이 무엇이며, 어떤 역할을 하는지 한 번 알아보도록 하겠습니다.

## vmlinux

우선 리눅스 커널은 다른 프로그램과 마찬가지로 하나의 실행 가능한 프로그램입니다. gcc를 통해 c언어 소스코드 `main.c`를 빌드했을 때 `a.out` 이름의 ELF 바이너리가 나오는 것 처럼 커널도 빌드하면 ELF 바이너리가 나오게 됩니다. 그리고 이 바이너리가 `vmlinux`입니다.

vmlinux는 기본적으로 정적 링크되어 있으며, 디버그 인포가 포함되어 있습니다. 또한 압축되어 있지 않습니다.

```
$ readelf -h vmlinux
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000080
  Start of program headers:          64 (bytes into file)
  Start of section headers:          65012128 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         5
  Size of section headers:           64 (bytes)
  Number of section headers:         38
  Section header string table index: 37
```

`vmlinux`는 사이즈가 크기 때문에 부트로더가 직접 로드할 수 없으며, 후에 알아볼 `vmlinuz`라는 형태로 압축해서 사용합니다. `vmlinux`는 주로 커널을 디버깅하는 용도로 사용하게 됩니다.

## vmlinuz / bzImage

`vmlinuz`는 `vmlinux`를 압축하여 부팅 가능하도록 만든 커널 이미지입니다. `/boot` 디렉토리를 보면 `vmlinuz`가 존재할 텐데, 바이오스가 `lilo` 혹은 `GRUB`과 같은 부트로더를 실행하고, 부트로더는 해당 이미지를 로드하여 커널을 실행하게 됩니다.

`vmlinuz`는 gzip 등으로 압축되어 약 15MB 정도의 크기를 가집니다. 또한 내부에 gzip의 압축 해제 코드가 내장되어 있어 런타임에 자가로 압축이 해제됩니다.

```shell
# /boot/grub/grub.cfg
...
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-ab65a1a2-0f26-4a66-a006-9c56630134a4' {
	recordfail
	load_video
	gfxmode $linux_gfx_mode
	insmod gzio
	if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
	insmod part_gpt
	insmod ext2
	set root='hd0,gpt2'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2  7d85336e-2019-4132-b599-e10bdb4f832b
	else
	  search --no-floppy --fs-uuid --set=root 7d85336e-2019-4132-b599-e10bdb4f832b
	fi
	linux	/vmlinuz-6.8.0-84-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro
	initrd	/initrd.img-6.8.0-84-generic
}
...
```
grub의 설정 파일을 보면 여러가지 부팅 옵션들을 볼 수 있는데, 그 중`linux` 옵션에 `vmlinuz-6.8.0-84-generic`를 지정해 주는 것을 볼 수 있습니다.

``` shell
make bzImage
```
`bzImage`는 `vmlinuz`의 원본이라고 보면 좋을 것 같습니다. 리눅스 커널을 위 명령어로 빌드하면 `arch/x86/linux/boot/`와 같은 디렉토리 아래에 `bzImage`가 생성됩니다.

``` shell
cp arch/x86/linux/boot/bzImage /boot/vmlinuz
```
이후에는 `bzImage`를 `vmlinuz`라는 이름으로 카피하여 사용합니다.   
   

### vmlinuz 로부터 vmlinux 추출하기
```shell
/usr/src/linux-headers-$(uname -r)/scripts/extract-vmlinux vmlinuz > vmlinux
```
커널이 제공하는 스크립트를 사용하여 `vmlinuz`로부터 `vmlinux`를 추출할 수 있습니다.

## 결론
이번 글에서는 `vmlinux`와 `vmlinuz`에 대해 알아보았습니다. 

정리하자면 vmlinux는 압축되지 않은 원본 ELF 커널 바이너리이고, `vmlinuz`/`bzImage`는 압축된 부팅 가능한 커널 이미지입니다. `vmlinux`는 주로 커널 개발자가 커널을 디버깅할 때 사용하며, `vmlinuz`는 부트로더가 실제로 로드하는 파일로, `/boot` 디렉토리에 저장되어 있습니다.

다음 글에서는 `initramfs`와 `initrd`에 대해 알아보도록 하겠습니다.
