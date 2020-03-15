# crosstool-ng_for_distcc
Build your own toolchain with crosstool-ng for Arch ARM volunteers to help build x86_64 stuff.

## Real speed gains compiling x86-64 can be realized with ARM volunteers
The folliwng examples illustrate that real and addititive gains can be realized when compiling x86-64 code using up to 2 RPi4B in the distcc cluster. The two compile tasks shown below used a x86-64 host alone, then the x86-64 host with 1x RPi4, then the x86-64 host with 2x RPi4s.

* Hostname mars was an Intel i7-4790k (quad core) running Arch Linux (x86_64).
* Hostname phobos was a Raspberry Pi 4B with 4G of RAM running Arch ARM (armv7h).
* Hostname deimos was a Raspberry Pi 4B with 2G of RAM running Arch ARM (armv7h).

### Linux kernel

Building kernel v4.19.111 (`make bzImage` only). The table below summarizes the runs compiling alone, or with each as noted.  The times shown are the mean of 4 runs.  The % change is relative to the time to compile on mars alone.  It is clear that using one ARM node decreases compilation time by ~7-8% and compiling with two ARM nodes gives an additive decrease in compilation time by ~15%.

| Nodes   | Time (sec)  | % change|
| -------- | :-----:| :----:|
| mars   |  185.3 ± 0.55 | -
| mars+phobos | 170.2 ± 0.91 | 8.2 ± 1.1
| mars+deimos|  171.6 ± 0.26  | 7.4 ± 0.6
| mars+phobos+deimos|  157.0 ± 0.79  | 15.2 ± 1.0

### Ffmpeg

Building ffmpeg v4.2.2, the table below summarizes the runs compiling alone, or with each as noted.  The times shown are the mean of 4 runs.  The % change is relative to the time to compile on mars alone.  It is clear that using one ARM node decreases compilation time by ~11-14% and compiling with two ARM nodes gives an additive decrease in compilation time by ~20%.

| Nodes   | Time (sec)  | % change|
| -------- | :-----:| :----:|
| mars   |  133.8 ± 0.29 | -
| mars+phobos | 117.98 ± 1.71 | 13.9 ± 1.7
| mars+deimos|  119.19 ± 2.42  | 10.9 ± 2.4
| mars+phobos+deimos|  106.38 ± 3.48  | 20.5 ± 3.5

## Download a pre-built package
I am supplying pre-built versions for armv7h and aarch64 as well as a corresponding systemd unit and config files in the [distccd-x86_64](https://aur.archlinux.org/packages/distccd-x86_64/) package in the [AUR](https://aur.archlinux.org/).  Feel free to grab them from there.  If you'd rather build your own, follow my crude steps below.

Note that just having the toolchain isn't enough, you'll also need to setup `distcc` on the ARM volunteers.  Again, see the package I linked above which does this (if your intent is not to use the package, you're probably savvy enough to setup the build but feel free to view the `PKGBUILD` as a guide).

## Build your own
### Setup
Building is required on the ARM device using the native architecture (armv7h or aarch64 for example). Below are some suggestions.

#### Thermal management
Depending on hardware, consider using active cooling to keep the device below its thermal throttle point.

#### Minimum reserved memory
Depending on hardware, increasing the following value can avoid some compilation errors due to reserved memory limitations with the kernel defaults:
```
echo 65536 > /proc/sys/vm/min_free_kbytes
```

Make it persistent across reboots by adding the following line to a file such as `/etc/sysctl.d/mine.conf`:
```
vm.min_free_kbytes=65536
```
#### Swapfile
Compiling a toolchain needs more memory than you think.  Even on a [RPi4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) with 4G of RAM, having a swap file can be needed to avoid throwing errors that will stop the build.

```
SWAPTARGET=/scratch/swapfile

dd if=/dev/zero of=$SWAPTARGET bs=1024 count=4194304
chmod 600 $SWAPTARGET
mkswap $SWAPTARGET
swapon $SWAPTARGET
```
### Building a toolchain with crosstool-ng
On the Arch ARM device, build and install the latest commit from [crosstool-ng-git](https://aur.archlinux.org/packages/crosstool-ng-git/).


Take the [config](https://github.com/graysky2/crosstool-ng_for_distcc/blob/master/config) file from this repo and run the build:

```
mkdir /scratch/tcbuild && cd /scratch/tcbuild
wget -O .config https://github.com/graysky2/crosstool-ng_for_distcc/blob/master/config
ct-ng build
```

You can optionally edit the `.config` I provided (or make your own entirely) running `ct-ng menuconfig`.  See the man page for more.

On my Rpi4 with 4G of RAM, it took 2h 22m to build the armv7h toolchain and 2h 36m to build the [aarch64](https://archlinuxarm.org/forum/viewtopic.php?f=67&t=14096) toolchain.

### Check for needed symlinks
Double check that the needed symlinks are present under `x-tools86/x86_64-pc-linux-gnu/bin` and create them if not.
```
 cd x-tools86/x86_64-pc-linux-gnu/bin
 mapfile -t filearr < <(find . -type f -printf '%P\n')
 for i in "${filearr[@]}"; do ln -s "$i" "${i/x86_64-pc-linux-gnu-/}" ; done
```

That will handle the executables but also check to see that the following symlink is present and make it if not:
`x86_64-pc-linux-gnu-cc -> x86_64-pc-linux-gnu-gcc`

### Helpful links
* https://archlinuxarm.org/wiki/Distcc_Cross-Compiling
