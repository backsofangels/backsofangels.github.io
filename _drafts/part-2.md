---
layout: post
title: "Repurposing an old tablet as ebook reader - Part 2"
description: "The Android 7.1.2 Nougat building hell in 2020s"
tags: [android, build, tablet, asus, nexus, nougat, aosp, android studio, kotlin, xml]
date: 2025-12-24
---

# 

Importante, ricordarsi di scrivere del cold boot dell'emulatore e del log

 .\emulator.exe -avd PixelXL -show-kernel -verbose > out.txt

comando per far partire l'emulatore

comando per far partire l'emulatore in cold boot invece, il flag è no snapshot

.\emulator.exe -avd PixelXL -no-snapshot -no-boot-anim -writable-system -show-kernel

.\emulator.exe -avd Nexus `
    -sysdir "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86" `
    -system "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\system.img" `
    -ramdisk "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\ramdisk.img" `
    -data "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\userdata.img" `
    -kernel "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\default\x86\kernel-ranchu" `
    -show-kernel `
    -no-snapshot-load `
    -verbose

.\emulator.exe -avd Nexus `
    -sysdir "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86" `
    -system "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\system.img" `
    -ramdisk "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\ramdisk.img" `
    -kernel "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\kernel-ranchu" `
    -show-kernel `
    -gpu off `
    -qemu -append "androidboot.gles=0 androidboot.hardware=goldfish vga=0"

.\qemu-system-i386.exe `
  -m 2048 `
  -cpu Nehalem `
  -accel whpx,kernel-irqchip=off `
  -kernel "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\default\x86\kernel-qemu" `
  -initrd "C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\default\x86\ramdisk.img" `
  -drive file="C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\system.img",format=raw,if=virtio `
  -drive file="C:\Users\backs\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\userdata.img",format=raw,if=virtio `
  -append "root=/dev/vda console=ttyS0 androidboot.hardware=goldfish qemu=1 androidboot.selinux=permissive" `
  -serial mon:stdio `
  -vga std `
  -net nic,model=virtio `
  -net user,hostfwd=tcp::5555-:5555 `
  -display sdl