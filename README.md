# pi-gen

Tool used to create WLAN Pi OS images. Our current GitHub setup automatically
builds a new image every day, or whenever new code is committed to the `force-build` branch.

To get more information on the complete build process used by pi-gen, please see
https://github.com/RPi-Distro/pi-gen/blob/master/README.md. Here is a summary for
most relevant details needed to customize the WLAN Pi OS image.

## Dependencies

pi-gen runs on Debian-based operating systems released after 2017, and we
always advise you use the latest OS for security reasons.

On other Linux distributions it may be possible to use the Docker build described
below.

To install the required dependencies for `pi-gen` you should run:

```bash
apt-get install coreutils quilt parted qemu-user-static debootstrap zerofree zip \
dosfstools libarchive-tools libcap2-bin grep rsync xz-utils file git curl bc \
gpg pigz xxd arch-test
```

The file `depends` contains a list of tools needed.  The format of this
package is `<tool>[:<debian-package>]`.

## Getting started with building your images

Getting started is as simple as cloning this repository on your build machine. You
can do so with:

```bash
git clone https://github.com/WLAN-Pi/pi-gen.git
```

`--depth 1` can be added afer `git clone` to create a shallow clone, only containing
the latest revision of the repository. Do not do this on your development machine.

Also, be careful to clone the repository to a base path **NOT** containing spaces.
This configuration is not supported by debootstrap and will lead to `pi-gen` not
running.

After cloning the repository, you can move to the next step and start configuring
your build.

## Config

Upon execution, `build.sh` will source the file `config` in the current
working directory.  This bash shell fragment is intended to set needed
environment variables.

You can configure several build options in this file. See more config the options on:
https://github.com/RPi-Distro/pi-gen/blob/master/README.md#config

The config file can also be specified on the command line as an argument the `build.sh` or `build-docker.sh` scripts.

```
./build.sh -c myconfig
```

This is parsed after `config` so can be used to override values set there.

This can help when making local changes that you don't want to merge officially.

## How the build process works

The following process is followed to build images:

 * Interate through all of the stage directories in alphanumeric order

 * Bypass a stage directory if it contains a file called
   "SKIP"

 * Run the script ```prerun.sh``` which is generally just used to copy the build
   directory between stages.

 * In each stage directory iterate through each subdirectory and then run each of the
   install scripts it contains, again in alphanumeric order. **These need to be named
   with a two digit padded number at the beginning.**
   There are a number of different files and directories which can be used to
   control different parts of the build process:

     - **00-run.sh** - A unix shell script. Needs to be made executable for it to run.

     - **00-run-chroot.sh** - A unix shell script which will be run in the chroot
       of the image build directory. Needs to be made executable for it to run.

     - **00-debconf** - Contents of this file are passed to debconf-set-selections
       to configure things like locale, etc.

     - **00-packages** - A list of packages to install. Can have more than one, space
       separated, per line.

     - **00-packages-nr** - As 00-packages, except these will be installed using
       the ```--no-install-recommends -y``` parameters to apt-get.

     - **00-patches** - A directory containing patch files to be applied, using quilt.
       If a file named 'EDIT' is present in the directory, the build process will
       be interrupted with a bash session, allowing an opportunity to create/revise
       the patches.

  * If the stage directory contains files called "EXPORT_NOOBS" or "EXPORT_IMAGE" then
    add this stage to a list of images to generate

  * Generate the images for any stages that have specified them

It is recommended to examine build.sh for finer details.


## Docker Build

Docker can be used to perform the build inside a container. This partially isolates
the build from the host system, and allows using the script on non-debian based
systems (e.g. Fedora Linux). The isolation is not complete due to the need to use
some kernel level services for arm emulation (binfmt) and loop devices (losetup).

To build:

```bash
vi config         # Edit your config file. See above.
./build-docker.sh
```

If everything goes well, your finished image will be in the `deploy/` folder.
You can then remove the build container with `docker rm -v pigen_work`

If you encounter errors during the build, you can edit the corresponding scripts, and
continue:

```bash
CONTINUE=1 ./build-docker.sh
```

To examine the container after a failure you can enter a shell within it using:

```bash
sudo docker run -it --privileged --volumes-from=pigen_work pi-gen /bin/bash
```

After successful build, the build container is by default removed. This may be undesired when making incremental changes to a customized build. To prevent the build script from remove the container add

```bash
PRESERVE_CONTAINER=1 ./build-docker.sh
```

There is a possibility that even when running from a docker container, the
installation of `qemu-user-static` will silently fail when building the image
because `binfmt-support` _must be enabled on the underlying kernel_. An easy
fix is to ensure `binfmt-support` is installed on the host machine before
starting the `./build-docker.sh` script (or using your own docker build
solution).

## Stage Anatomy

### WLAN Pi Stage Overview

The build of Raspbian is divided up into several stages for logical clarity
and modularity.  This causes some initial complexity, but it simplifies
maintenance and allows for more easy customization.

 - **Stage 0** - bootstrap.  The primary purpose of this stage is to create a
   usable filesystem.  This is accomplished largely through the use of
   `debootstrap`, which creates a minimal filesystem suitable for use as a
   base.tgz on Debian systems.  This stage also configures apt settings and
   installs `raspberrypi-bootloader` which is missed by debootstrap.  The
   minimal core is installed but not configured. As a result, this stage will not boot.

 - **Stage 1** - truly minimal system.  This stage makes the system bootable by
   installing system files like `/etc/fstab`, configures the bootloader, makes
   the network operable, and installs packages like raspi-config.  At this
   stage the system should boot to a local console from which you have the
   means to perform basic tasks needed to configure and install the system.

 - **Stage 2** - lite system.  This stage produces the Raspberry Pi OS Lite image.
   Stage 2 installs some optimized memory functions, sets timezone and charmap
   defaults, installs fake-hwclock and ntp, wireless LAN and bluetooth support,
   dphys-swapfile, and other basics for managing the hardware.  It also
   creates necessary groups and gives the pi user access to sudo and the
   standard console hardware permission groups.

   Note: Raspberry Pi OS Lite contains a number of tools for development,
   including `Python`, `Lua` and the `build-essential` package. If you are
   creating an image to deploy in products, be sure to remove extraneous development
   tools before deployment.

 - **Stage wlanpi 0** - base WLAN Pi packages. This stage installs the base
   packages for the WLAN Pi customization. Generating an image here is not
   expected nor tested.

 - **Stage wlanpi 1** - full WLAN Pi customization. This stage completes the
   WLAN Pi customization setup, configuring services, permissions and several
   other system level customization.
   
 - **Stage wlanpi hwtest** - include HW tests. This stage installs packages
   needed to run hardware tests on the WLAN Pi Pro board. This is mainly
   inteded for factory tests to ensure the boards are funcitonal before shipping.
   This stage is not built by default - add `wlanpi_hwtest` to `STAGE_LIST` on
   `config` file to build its image.

 - **Stage 3/4/5** - desktop/full image. Those stages are used by Raspbian to
   create the desktop and full image. Since WLAN Pi isn't supposed to be used
   as a full desktop system, those stages are not used.


## Skipping stages to speed up development

If you're working on a specific stage the recommended development process is as
follows:

 * Add a file called SKIP_IMAGES into the directories containing EXPORT_* files
   (currently wlanpi1 and wlanpi_hwtest)
 * Run build.sh to build all stages
 * Add SKIP files to the earlier successfully built stages
 * Modify the last stage
 * Rebuild just the last stage using ```sudo CLEAN=1 ./build.sh```
 * Once you're happy with the image you can remove the SKIP_IMAGES files and
   export your image to test

# Troubleshooting

## `64 Bit Systems`
Please note there is currently an issue when compiling with a 64 Bit OS. See
https://github.com/RPi-Distro/pi-gen/issues/271

A 64 bit image can be generated from the `arm64` branch in this repository. Just
replace the command from [this section](#getting-started-with-building-your-images)
by the one below, and follow the rest of the documentation:
```bash
git clone --branch arm64 https://github.com/RPI-Distro/pi-gen.git
```

If you want to generate a 64 bits image from a Raspberry Pi running a 32 bits
version, you need to add `arm_64bit=1` to your `config.txt` file and reboot your
machine. This will restart your machine with a 64 bits kernel. This will only
work from a Raspberry Pi with a 64-bit capable processor (i.e. Raspberry Pi Zero
2, Raspberry Pi 3 or Raspberry Pi 4).


## `binfmt_misc`

Linux is able execute binaries from other architectures, meaning that it should be
possible to make use of `pi-gen` on an x86_64 system, even though it will be running
ARM binaries. This requires support from the [`binfmt_misc`](https://en.wikipedia.org/wiki/Binfmt_misc)
kernel module.

You may see one of the following errors:

```
update-binfmts: warning: Couldn't load the binfmt_misc module.
```
```
W: Failure trying to run: chroot "/pi-gen/work/test/stage0/rootfs" /bin/true
and/or
chroot: failed to run command '/bin/true': Exec format error
```

To resolve this, ensure that the following files are available (install them if necessary):

```
/lib/modules/$(uname -r)/kernel/fs/binfmt_misc.ko
/usr/bin/qemu-aarch64-static
```

You may also need to load the module by hand - run `modprobe binfmt_misc`.

If you are using WSL to build you may have to enable the service `sudo update-binfmts --enable`

# Contributing

Got some improvements or fixes you'd like to see in the image? Please feel free to
open an issue and pull request. We have some general guidelines on [Contributing](https://github.com/WLAN-Pi/.github/blob/main/contributing.md).

If in doubt, open an issue anyway and let's discuss it!
