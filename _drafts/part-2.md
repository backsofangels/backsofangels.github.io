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

  ripartiamo da zero con build però stavolta aosp

quidni lancio il source build/envtralala.sh
make clobber per pulire out
lunch aosp_x86-eng
build-android (mio alias per m -j$(nproc) )

"Riallineamento del Fstab per Architetture a Blocchi:" "La presenza di riferimenti mtdblock nel file fstab.goldfish è incompatibile con l'emulazione QEMU moderna e con i target basati su eMMC come Tilapia. La migrazione verso i nodi /dev/block/vdX risolve il fallimento del mount del filesystem ext4 durante la fase di init."

Analisi per QEMU (Situazione attuale)
Per far funzionare la build su QEMU sotto Windows, questa configurazione non è corretta. QEMU emula dischi rigidi (IDE o VirtIO) che vengono visti dal kernel come /dev/block/vda o /dev/block/sda. Finché il file fstab punta a mtdblock, il mount fallirà sistematicamente.

Analisi per Tilapia (Nexus 7 2012 3G)
In merito alla tua domanda sulla futura build per Tilapia: questa configurazione non va bene.

Il Nexus 7 (Tilapia/Grouper) non utilizza memorie MTD (NAND Flash raw), ma utilizza una memoria eMMC. Nei dispositivi Android basati su eMMC, le partizioni vengono mappate come unità a blocchi. Nello specifico, per il Nexus 7, i percorsi nel file fstab dovrebbero puntare ai nodi gestiti dal controller SDHCI, solitamente strutturati in questo modo:

/dev/block/platform/sdhci-tegra.3/by-name/system

Procedura di correzione per la build attuale
Prima di procedere con make clobber, è necessario modificare device/generic/goldfish/fstab.goldfish per adattarlo all'emulazione moderna. Sostituisci il contenuto con i seguenti parametri:

Plaintext

# <src>                   <mnt_point> <type>  <mnt_flags and options> <fs_mgr_flags>
/dev/block/vda            /system     ext4    ro,barrier=1            wait
/dev/block/vdb            /data       ext4    noatime,nosuid,nodev    wait,check
/dev/block/vdc            /cache      ext4    noatime,nosuid,nodev    wait,check
Nota tecnica: L'utilizzo di vda, vdb e vdc presuppone l'uso dei driver VirtIO in QEMU, che garantiscono prestazioni superiori e una gestione dei blocchi più stabile su Windows 11.

Sintesi dei passaggi operativi
Modifica il file: Applica i cambiamenti sopra indicati a device/generic/goldfish/fstab.goldfish.

Esegui la pulizia: make clobber.

Configura l'ambiente: source build/envsetup.sh.

Seleziona il target: lunch aosp_x86-eng.

Avvia la compilazione: make -j$(nproc).

ho fatto un backup prima perchè si in fstab_goldfish_bck

importante!!!: l'export delle configurazioni jack

La differenza tra `mini-emulator-x86` e `aosp_x86` risiede nella finalità della build e nel set di driver e binari inclusi. Sebbene entrambi puntino all'architettura Intel, la loro struttura interna è ottimizzata per scenari d'uso differenti.

### 1. Target `mini-emulator-x86`

Questo target è progettato per test di integrazione minimi e per lo sviluppo del framework Android stesso.

* **Contenuto:** Include solo i componenti essenziali per avviare il sistema. Molte applicazioni standard di sistema (come la Galleria o il Browser) vengono rimosse per accelerare i tempi di compilazione.
* **Driver:** È configurato per un hardware virtuale estremamente semplificato. Come hai riscontrato nel file `fstab`, spesso utilizza ancora mappature legacy come `mtdblock`, poiché è pensato per essere eseguito all'interno dell'ambiente di test continuo di Google (Continuous Integration).
* **Kernel:** Viene spesso compilato con una configurazione ridotta all'osso che potrebbe non includere i driver per le periferiche di input moderne o per l'accelerazione VirtIO.

### 2. Target `aosp_x86` (o `sdk_phone_x86`)

Questo è il target standard per lo sviluppo di applicazioni e per la distribuzione dell'emulatore ufficiale di Android Studio.

* **Contenuto:** Include l'intera suite di applicazioni AOSP e i servizi di sistema necessari per un'esperienza utente completa. È la base ideale per il tuo progetto di e-reader, poiché offre un sistema operativo "finito".
* **Driver:** È ottimizzato per l'architettura **Goldfish/Ranchu**. Supporta nativamente le estensioni VirtIO per dischi e rete e include i driver grafici necessari per interfacciarsi correttamente con QEMU su sistemi host moderni.
* **Compatibilità:** Utilizza tabelle di mount basate su unità a blocchi (`/dev/block/vda`), eliminando i conflitti legati ai vecchi driver NAND flash.

---

### Tabella Comparativa Tecnica

| Caratteristica | mini-emulator-x86 | aosp_x86 |
| --- | --- | --- |
| **Finalità** | Test unitari / CI | Sviluppo App / Emulazione completa |
| **Set App** | Minimale (Barebone) | Standard AOSP |
| **Gestione Dischi** | Spesso MTD (Legacy) | Block Devices (Moderno) |
| **Kernel** | Ridotto | Completo (Goldfish) |
| **Stabilità in QEMU** | Bassa (richiede tweak manuali) | Alta (Plug & Play) |

---

### Nota per il Diario Tecnico

> **"Selezione del Target Architetturale:"**
> *"Il passaggio da `mini-emulator-x86` a `aosp_x86-eng` è necessario per allineare l'output della build agli standard di emulazione moderni. Mentre il primo è un ambiente di test sintetico, il secondo fornisce lo stack completo di driver e applicazioni richiesto per un deployment funzionale su QEMU Windows 11."*

Se hai completato la risoluzione degli errori del server Jack, la compilazione con il target `aosp_x86` dovrebbe procedere senza i problemi di mount riscontrati precedentemente.

Primo test con QEMU, comando lanciato

.\qemu-system-i386.exe `
  -m 2048 `
  -cpu Nehalem `
  -accel whpx,kernel-irqchip=off `
  -kernel "C:\Users\backs\generic_x86\kernel-qemu" `
  -initrd "C:\Users\backs\generic_x86\ramdisk.img" `
  -drive file="C:\Users\backs\generic_x86\system.img",format=raw,if=none,id=system `
  -device virtio-blk-pci,drive=system `
  -drive file="C:\Users\backs\generic_x86\userdata.img",format=raw,if=none,id=userdata `
  -device virtio-blk-pci,drive=userdata `
  -append "androidboot.hardware=goldfish androidboot.selinux=permissive console=tty0" `
  -vga std `
  -display sdl

.\qemu-system-i386.exe `
  -m 2048 `
  -cpu Nehalem `
  -accel whpx,kernel-irqchip=off `
  -kernel "C:\Users\backs\generic_x86\kernel-qemu" `
  -initrd "C:\Users\backs\generic_x86\ramdisk_sata.img" `
  -drive file="C:\Users\backs\generic_x86\system.img",index=0,media=disk,format=raw `
  -drive file="C:\Users\backs\generic_x86\userdata.img",index=1,media=disk,format=raw `
  -append "androidboot.hardware=goldfish qemu=1 androidboot.selinux=permissive console=tty0" `
  -vga virtio -device virtio-gpu-pci `
  -display sdl -net nic -net user,hostfwd=tcp::5555-:5555
  