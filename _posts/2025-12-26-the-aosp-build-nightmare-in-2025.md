---
title: The AOSP building nightmare in 2025
description: The trial-and-error process of building the AOSP 7.1.2
date: 2025-12-26T09:52:38.427Z
draft: true
tags:
    - android
    - aosp
    - build
    - ereader
    - linux
    - make
    - tablet
    - ebook
slug: aosp-building-nightmare-2025
---

## Preface

Following the part 1, the next meaningful step is to start working towards a custom AOSP build, based on Nougat 7.1.2, the latest version available for the Nexus 7 codename Tilapia.

Of course, no one wants to flash a just-built ROM on the only device available, so before getting to that step it was important to test the build locally, in an emulator. The choice obviously went toward the built-in Android Studio emulator, which is more or less the standard for this kind of stuff, and to the core it's nothing but a hardened QEMU version.

At this point I was ready to build, clone, make, test, be happy!

**No.**

Let's get to how I spent two days compiling, making and cursing to QEMU for two days straight, after understanding I choose the completely wrong build target and recompiling everything for *three* times.

### The build environment setup

First of all, the foundational thing to do is to install a WSL2 ecosystem, I will never thank Microsoft enough for this little gem, saving me the hassle of dual booting to Linux. 

The main concern is the slowness of I/O between the WSL2 machine and the Windows system mount it does, but it's completely ignorable if the build happens completely in the WSL2 ecosystem and the files are just copied to Windows mount once done.

Keeping in consideration the quite old AOSP version, Ubuntu v20.04 seems a nice fit, with all the needed packages

{% highlight bash %}
sudo apt update && sudo apt upgrade -y && sudo apt install -y \
  git-core \
  gnupg \
  flex \
  bison \
  build-essential \
  zip \
  curl \
  zlib1g-dev \
  gcc-multilib \
  g++-multilib \
  libc6-dev-i386 \
  libncurses5 \
  libncurses5-dev \
  x11proto-core-dev \
  libx11-dev \
  libgl1-mesa-dev \
  libxml2-utils \
  xsltproc \
  unzip \
  fontconfig \
  openjdk-8-jdk
{% endhighlight %}

There's quite some stuff to be installed to be sure everything compiles correctly, and that's only the environment preparation. 

**Important note:** if you have multiple java installations, just choose Java 8 like this

{% highlight bash %}
sudo update-alternatives --config java
sudo update-alternatives --config javac
{% endhighlight %}

After this, let's just *mkdir android* to create the working directory, to keep everything nice and clean. Now, it's time to install the [repo](https://gerrit.googlesource.com/git-repo) tool.

{% highlight bash %}
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
{% endhighlight %}

At this point, proceeding with the clone of the repo is the next thing to do, but *which repo*? My first thought is to go with LineageOS but, obviously, the code isn't available anymore. But, do I really need LineageOS?

Short answer: no.

Long answer: *no, but in italics*.

Jokes aside, the tablet itself has mounted LineageOS right now and it's quite clunky and slow. AOSP seems like a good candidate considering also the modifications I will apply in the future, so let's clone that one instead, including the kernel.

{% highlight bash %}
cd android
repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android-7.1.2_r36
repo sync -c --no-tags -j8
{% endhighlight %}

I'll keep the part of the vendor blobs for later, we're just building towards an emulator, so it's possible to go straight at it.

## The platform nightmare

After all this stuff, it's possible to start the build. All easy, just two commands, magic happens, boom, done.

*Hahahahaha.*

First of all, to launch the build itself this two commands are needed

{% highlight bash %}
source build/envsetup.sh
lunch
{% endhighlight %}

At this point, lunch gives you various options, that's where I did my first mistake.

{% highlight bash %}
user@WSL:~/android$ lunch

You're building on Linux

Lunch menu... pick a combo:
    1. aosp_arm-eng
    2. aosp_arm64-eng
    3. aosp_mips-eng
    4. aosp_mips64-eng
    5. aosp_x86-eng
    6. aosp_x86_64-eng
    7. full_fugu-userdebug
    8. aosp_fugu-userdebug
    9. aosp_grouper-userdebug
    10. aosp_tilapia-userdebug
    11. full_tilapia-userdebug
    12. mini_emulator_arm64-userdebug
    13. m_e_arm-userdebug
    14. m_e_mips-userdebug
    15. m_e_mips64-eng
    16. mini_emulator_x86-userdebug
    17. mini_emulator_x86_64-userdebug # --> The culprit is here <--
    18. aosp_dragon-userdebug
    19. aosp_dragon-eng
    20. aosp_marlin-userdebug
    21. aosp_sailfish-userdebug
    22. aosp_flounder-userdebug
    23. aosp_angler-userdebug
    24. aosp_bullhead-userdebug
    25. hikey-userdebug
    26. aosp_shamu-userdebug
{% endhighlight %}

Not having much knowledge of this, I went for the mini_emulator_x86_64-userdebug build. Now, let's say it's a bit misleading but as far as I understood, that target is more oriented towards a CI/CD testing, since it's a very very minimal Android distribution. Nevertheless, let's see what happens if you try to run this in an emulator.

{% highlight bash %}
source build/envsetup.sh
lunch mini_emulator_x86-userdebug
m -j$(nproc)

...

error: ro.build.fingerprint cannot exceed 91 bytes: Android/mini_emulator_x86_64/mini-emulator-x86_64:7.1.2/N2G48H/wslus12231940:userdebug/test-keys (96)
[ 35% 13695/38461] target C++: libLLVMCore <= external/llvm/lib/IR/Function.cpp ninja: build stopped: subcommand failed. make: *** [build/core/ninja.mk:149: ninja_wrapper] Error 1
{% endhighlight %}

Okay, so seems like the generated build fingerprint is too long, exceeding the 91 bytes assigned by *five bytes*. At this point, we need to modify the *build/tools/buildinfo.sh* to cut down the string, like this

{% highlight bash %}
ro.build.fingerprint=$(echo $BUILD_FINGERPRINT | cut -c1-91)
rm -rf out/target/product/mini-emulator-x86_64/obj/ETC/system_build_prop_intermediates
{% endhighlight %}

Nevertheless, even after this modification the build kept on blocking itself, so let's just do a more deep intervention: the key point is to modify the device/generic/mini_emulator_x86_64/mini_emulator_x86_64.mk in such a way the product names and details are shorter, obtaining the following

{% highlight bash %}
$(call inherit-product, device/generic/x86_64/mini_x86_64.mk)

$(call inherit-product, device/generic/mini-emulator-armv7-a-neon/mini_emulator_common.mk)

PRODUCT_NAME := mini_x64
PRODUCT_DEVICE := mini_x64
PRODUCT_BRAND := AOSP
PRODUCT_MODEL := mini_x64

LOCAL_KERNEL := prebuilts/qemu-kernel/x86_64/kernel-qemu
PRODUCT_COPY_FILES += \
    $(LOCAL_KERNEL):kernel
{% endhighlight %}

and renaming the folder to mini_x64 for coherence. After sourcing again the build env, the same target that was before named as mini_emulator_x86_64 now appears as mini_x64. This was not enough anyway, since the issues now turned out to be a memory address issue present in WSL with the ART compiler. The fix for this is to set the flag WITH_DEXPREOPT = false in the device BoardConfig.mk file. Even after this, anyway, Jack kept crashing.

{% highlight bash %}
Launching Jack server java -XX:MaxJavaStackTraceDepth=-1 -Djava.io.tmpdir=/tmp -Dfile.encoding=UTF-8 -XX:+TieredCompilation -cp /home/wsl/.jack-server/launcher.jar com.android.jack.launcher.ServerLauncher Jack server failed to (re)start, try 'jack-diagnose' or see Jack server log SSL error when connecting to the Jack server. Try 'jack-diagnose' SSL error when connecting to the Jack server. Try 'jack-diagnose' 
[ 50% 13360/26200] target C++: oatdump <= art/oatdump/oatdump.cc ninja: build stopped: subcommand failed. make: *** [build/core/ninja.mk:149: ninja_wrapper] Error 1
{% endhighlight %}

Searching around, seems like re-enabling older TLS versions for jack communication would do the trick, so we need to modify how the JVM security behaves, removing TLSv1 and TLSv1.1 from the disabled algorithms. At this point, just because we're at it, let's add also a line to our .bashrc to give jack some more memory

{% highlight bash %}

sudo nano /usr/lib/jvm/java-8-openjdk-amd64/jre/lib/security/java.security

... searching for jdk.tls.disabledAlgorithms ...

... removing TLSv1 and TLSv1.1

echo 'export JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4g"' >> ~/.bashrc
source ~/.bashrc

cd android/prebuilts/sdk/tools/jack-admin
kill-server
start-server

cd ~/android
make clean
lunch mini_x64-userdebug
m -j$(nproc)

{% endhighlight %}

Now, the build was successful and everything went straight to the end, after 20 minutes more or less.

## The testing phase, or the Nine Hells of Baator

The *.img files are ready, kernel is there, everything seems in place, time to spin up the emulator. First thing that comes in mind is to start using the AVDs from Android Studio, so after creating a custom AVD inside the GUI named Nexus, built upon a PixelXL device, and launched in the following way

{% highlight powershell %}
.\emulator.exe -avd Nexus `
    -sysdir "$env:USERPROFILE\AppData\Local\Android\Sdk\system-images\android-25\custom-x86" `
    -system "$env:USERPROFILE\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\system.img" `
    -ramdisk "$env:USERPROFILE\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\ramdisk.img" `
    -data "$env:USERPROFILE\AppData\Local\Android\Sdk\system-images\android-25\custom-x86\userdata.img" `
    -kernel "$env:USERPROFILE\AppData\Local\Android\Sdk\system-images\android-25\default\x86\kernel-ranchu" `
    -show-kernel `
    -no-snapshot-load `
    -verbose
{% endhighlight %}

Of course the emulator didn't started at all and the logs aren't encouraging, neither they were clear.

First thing to do was to understand if it was a GPU compatibility, which I tested adding the following combination of flags

{% highlight powershell %}
-gpu off

-gpu swiftshader_indirect -qemu -append "androidboot.gles=0 androidboot.hardware=goldfish vga=0"
{% endhighlight %}

The main point was an error saying "Bad color buffer handle", which happened to be present in almost every try I did. Same went even after compiling the image for mini_emulator_x86, which I tested with raw QEMU as well, for a whole day. So, I resorted to the last thing to do: build an aosp_x86-eng.

# The light at the end of the tunnel

First thing is to check if the mount points for the goldfish kernel are right: usually, it's like this

{% highlight bash %}
\# Android fstab file.
\#<src>                                                  <mnt\_point>         <type>    <mnt\_flags and options>                              <fs\_mgr\_flags>

\# The filesystem that contains the filesystem checker binary (typically /system) cannot

\# specify MF\_CHECK, and must come before any filesystems that do specify MF\_CHECK

/dev/block/mtdblock0                                    /system             ext4      ro,barrier=1                                         wait
/dev/block/mtdblock1                                    /data               ext4      noatime,nosuid,nodev,barrier=1,nomblk\_io\_submit      wait,check
/dev/block/mtdblock2                                    /cache              ext4      noatime,nosuid,nodev  wait,check
/devices/platform/goldfish\_mmc.0\*                     auto                auto      defaults                                             voldmanaged=sdcard:auto,encryptable=userdata
{% endhighlight %}

But QEMU and older 

So, let's start fresh:
- check if `device/generic/goldfish/fstab.goldfish` has 
- `make clobber` to clean everything
- `source build/envsetup.sh` to pick up configuration
- `lunch aosp_x86-eng` to populate variables
- `m -j$(nproc)` to actually build