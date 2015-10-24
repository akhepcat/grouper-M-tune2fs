The basic method to build the tune2fs tool follows the same directions as building the actual ROM,
as described in the original post: http://dmitry.gr/index.php?r=06.%20Thoughts&proj=04.%20Android%20M%20on%20Grouper
and further discussed here: http://forum.xda-developers.com/nexus-7/development/rom-aosp-6-0-grouper-t3225584/


This is very space intensive, and you will need approximately 100Gb of free disk space
in order to build the ROM.

Also, you'll need about a day to get things done, depending on the speed of your network
connection and the speed of your compiling computer.

### Begin script:

#get android M AOSP into folder called M
mkdir M
cd M
repo init -u https://android.googlesource.com/platform/manifest -b android-6.0.0_r1
repo sync -j4
cd ..

#get latest L AOSP into folder called L
cd L
repo init -u https://android.googlesource.com/platform/manifest -b android-5.1.1_r1
repo sync -j4
cd ..

#get blobs
cd M
wget https://dl.google.com/dl/android/aosp/asus-grouper-lmy47v-f395a331.tgz
wget https://dl.google.com/dl/android/aosp/broadcom-grouper-lmy47v-5671ab27.tgz
wget https://dl.google.com/dl/android/aosp/elan-grouper-lmy47v-6a10e8f3.tgz
wget https://dl.google.com/dl/android/aosp/invensense-grouper-lmy47v-ccd43018.tgz
wget https://dl.google.com/dl/android/aosp/nvidia-grouper-lmy47v-c9005750.tgz
wget https://dl.google.com/dl/android/aosp/nxp-grouper-lmy47v-18820f9b.tgz
wget https://dl.google.com/dl/android/aosp/widevine-grouper-lmy47v-e570494f.tgz
tar xvfz asus-grouper-lmy47v-f395a331.tgz
tar xvfz broadcom-grouper-lmy47v-5671ab27.tgz
tar xvfz elan-grouper-lmy47v-6a10e8f3.tgz
tar xvfz invensense-grouper-lmy47v-ccd43018.tgz
tar xvfz nvidia-grouper-lmy47v-c9005750.tgz
tar xvfz nxp-grouper-lmy47v-18820f9b.tgz
tar xvfz widevine-grouper-lmy47v-e570494f.tgz
dd if=extract-asus-grouper.sh bs=14466 skip=1       | tar xvz
dd if=extract-broadcom-grouper.sh bs=14464 skip=1   | tar xvz
dd if=extract-elan-grouper.sh bs=14490 skip=1       | tar xvz
dd if=extract-invensense-grouper.sh bs=14456 skip=1 | tar xvz
dd if=extract-nvidia-grouper.sh bs=14460 skip=1     | tar xvz
dd if=extract-nxp-grouper.sh bs=14452 skip=1        | tar xvz
dd if=extract-widevine-grouper.sh bs=14446 skip=1   | tar xvz
rm *grouper-lmy47v*.tgz extract-*-grouper.sh

#cool binary patch for GL blobs
echo -n dmitrygr_libldr | dd bs=1 seek=4340 conv=notrunc of=vendor/nvidia/grouper/proprietary/libEGL_tegra.so
echo -n dgv1 | dd bs=1 seek=6758 conv=notrunc of=vendor/nvidia/grouper/proprietary/libEGL_tegra.so
echo -n dmitrygr_libldr | dd bs=1 seek=3811 conv=notrunc of=vendor/nvidia/grouper/proprietary/libGLESv1_CM_tegra.so
echo -n dgv1 | dd bs=1 seek=6447 conv=notrunc of=vendor/nvidia/grouper/proprietary/libGLESv1_CM_tegra.so

#cool binary patch for GPS blob
printf "malloc\0" | dd bs=1 seek=5246 conv=notrunc of=vendor/broadcom/grouper/proprietary/glgps

#get old device sources
cp -Rvf ../L/device/asus/grouper device/asus/grouper

#apply source patch to Nfc package (sadly we must mess with platform code here)
cd packages/apps/Nfc/
git apply ../../../../packages-apps-Nfc.patch
cd ../../..

#apply source patch to vendor repo
cd vendor
git apply ../../vendor.patch
cd ..

#apply source patch to device repo
cd device/asus/grouper
git apply ../../../../device-asus-grouper.patch
cd ../../..

#get kernel
cd ..
git clone https://android.googlesource.com/kernel/tegra.git
cd tegra
git checkout remotes/origin/android-tegra3-grouper-3.1-lollipop-mr1 -b l-mr1

#apply kernel patch
git apply ../kernel.patch

#build kernel
export CROSS_COMPILE=`pwd`/../M/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8/bin/arm-eabi-
export ARCH=arm
make tegra3_android_defconfig
make -j4
cp arch/arm/boot/zImage ../M/device/asus/grouper/kernel
cd ../M

#build Android
source build/envsetup.sh
lunch aosp_grouper-userdebug
make -j4

# Now make tune2fs
make ./out/target/product/grouper/symbols/system/bin/tune2fs

