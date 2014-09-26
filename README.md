ev3dev-buildscripts
===================

These are the scripts used to compile the ev3dev kernel. Originally it also
included scripts to bootstrap a root file system and create a disk image.
Those scripts have evolved into the [brickstrap] package.


System Requirements
-------------------
* Debian or derivative OS (Ubuntu, Mint, etc.)
* User account with `sudo` enabled

Scripts
-------

`build-kernel`               Used to build the kernel.

`defconfig`                  Used to manage the ev3dev_defconfig file and
                             your current local configuration.

`install-kernel-build-tools` Installs the prerequisite tools required
                             to build the kernel.

`menuconfig`                 Runs the menu configuration tool for the
                             kernel configuration.


First time kernel build
-----------------------

1.  If you don't have `git` already, then we need to install it.

        ~ $ sudo apt-get install git

2.  Create a working directory somewhere. For this tutorial, we are using
    `~/work`. The build scripts will generate extra subdirectories here
    so we suggest creating a new directory instead of using an existing one.

        ~ $ mkdir work
        ~ $ cd work

3.  Clone the kernel and root file system repos.

        ~/work $ git clone git://github.com/ev3dev/ev3dev-kernel
        ~/work $ git clone git://github.com/ev3dev/ev3dev-buildscripts

4.  Change to the `ev3dev-buildscripts` directory and have a look around.

        ~/work $ cd ev3dev-buildscripts
        ~/work/ev3dev-buildscripts $ ls
        boot.cmd        build-kernel  install-kernel-build-tools  local-env   README.md
        build-boot-scr  defconfig     LICENSE                     menuconfig  setup-env


5.  Now we need to install the required tool. To do this, we need to add the
    ev3dev package repo and then run the `install-kernel-build-tools` script.
    (You only need to run this once.)

        ~/work/ev3dev-buildscripts $ .sudo apt-add-repository http://ev3dev.org/debian
        ~/work/ev3dev-buildscripts $ .sudo apt-key adv --keyserver pgp.mit.edu --recv-keys 2B210565
        ~/work/ev3dev-buildscripts $ .sudo apt-get update
        ~/work/ev3dev-buildscripts $ ./install-kernel-build-tools

6.  Create a `local-env` to make use of all of your processing power. See the
    [Faster Builds and Custom Locations](#faster-builds-and-custom-locations)
    section below for more about this file.

        ~/work/ev3dev-buildscripts $ echo "#!/bin/sh
        
        export EV3DEV_MAKE_ARGS=-j4" > local-env

7.  Now we can compile the kernel.

        ~/work/ev3dev-buildscripts $ ./build_kernel

8.  That's it! The uImage and kernel modules you just built are saved in
    `../dist`. You just need to copy the files to your
    already formatted SD card. For an easier way of getting the kernel on
    your EV3, see [Sharing Your Kernel](#sharing-your-kernel).

        ~/work/ev3dev-buildscripts $ cd ../dist
        ~/work/dist $ cp uImage <path-to-boot-partition>/uImage
        ~/work/dist $ sudo cp -r lib/ <path-to-file-system-partition>


Faster Builds and Custom Locations
----------------------------------

By default the locations of the kernel source tree and the toolchain used
to build the kernel are expected to be in certain directories relative to
the ev3dev-buildscripts repo directory.

You can override these locations by creating a file called `local-env`
in the ev3dev-buildscripts directory or `~/.ev3dev-env` (in your home directory).
It should look like this:

    #!/bin/sh
    
    export EV3DEV_MAKE_ARGS=-j4
    
    # override any EV3DEV_* variables from setup-env script.
    #export EV3DEV_XXX=/custom/path
    #export EV3DEV_MERGE_CMD="kdiff3 \$file1 \$file2"
    #export EV3DEV_MERGE_CMD="meld \$file1 \$file2"

The `-j4` is for faster builds. It allows make to compile files in
in parallel. You should replace 4 with the number of processor cores that
you want to devote to building the kernel.

You can use custom paths to make the `build-kernel` script automatically
install the kernel and modules directly on the EV3! First, you need to
mount the EV3 root file system. You can use nfs or sshfs (check the
[wiki] on how to do this). Then just set the appropriate paths in your
`local-env` like this:

    # replace `/mnt/ev3dev-root` with your actual mount point
    export EV3DEV_INSTALL_KERNEL=/mnt/ev3dev-root/media/mmc_p1
    export EV3DEV_INSTALL_MODULES=/mnt/ev3dev-root


Managing the Kernel Configuration
---------------------------------

When you run `./build-kernel` if no existing kernel configuration exists
the default configuration is loaded from `arch/arm/configs/ev3dev_defconfig`.

If you make changes to your local kernel configuration that you want to merge
into the default configuration, run `./defconfig update`. It will use the
merge tool specified by the EV3DEV_MERGE_CMD environment variable.

If you have an existing kernel configuration, you will want to check for changes
to the default configuration each time you merge or checkout a branch. You can
call `./defconfig load` to wipe out your local configuration and load the
default configuration or you can call `./defconfig merge` to merge the
default configuration into your existing local configuration.

If you are forgetful or lazy or just want this to happen automatically, you can
set up hooks in your git repo. For example, you could save the following file as
both `.git/hooks/post-merge` and `.git/hooks/post-checkout` and you will
be prompted to merge the default configuration into your local configuration
whenever you merge or checkout a branch. In you followed the tutorial above,
`<path-to-ev3dev-buildscripts-repo>` would be `~/work/ev3dev-buildscripts`.

    #!/bin/sh
    
    <path-to-ev3dev-buildscripts-repo>/defconfig merge


Sharing Your Kernel
-------------------

Want to send your custom kernel to someone so that they can use it? Never fear,
there is an easy way to do that - using Debian packaging.

First, we want to set a kernel option so that our friends will know what kernel
they are running. Run `./menuconfig` and set this option:

    General setup --->
      (-your-name) Local version - append to kernel release

Make sure to include the '-' prefix in `-your-name` on the _Local version_.
And, of course, substitute something like your github user name for _your-name_.

Then, we build a Debian package.

    ~/work/ev3dev-buildscripts $ ./build-kernel deb-pkg KDEB_PKGVERSION=1
    ...
    <lots-of-build-output>
    ...
    ~/work/ev3dev-buildscripts $ ls ../*.deb
    ../linux-headers-3.3.0-3-ev3dev-your-name_1_armel.deb
    ../linux-image-3.3.0-3-ev3dev-your-name_1_armel.deb
    ../linux-libc-dev_1_armel.deb

Now, send the `linux-image-*` file to your friend with these instructions:

* Copy the `.deb` file to your EV3
* Install the package
* Reboot the EV3

Example:

    user@host ~ $ scp linux-image-*.deb root@ev3dev:/tmp
    user@host ~ $ ssh root@ev3dev
    root@ev3dev:~# dpkg --install /tmp/linux-image-*.deb
    root@ev3dev:~# reboot

Common Errors
-------------

* If you see this error...

        ERROR: ld.so: object 'libfakeroot-sysv.so' from LD_PRELOAD cannot be preloaded (wrong ELF class: ELFCLASS64): ignored.
    
    ...just ignore it. It is normal (a side effect of cross-compiling).

* If you see an error related to `asm/bitsperlong.h` like this:

        ...
          Generating include/generated/mach-types.h
          CC      kernel/bounds.s
        In file included from /home/user/ev3dev-kernel/arch/arm/include/asm/types.h:4:0,
                          from /home/user/ev3dev-kernel/include/linux/types.h:4,
                         from /home/user/ev3dev-kernel/include/linux/page-flags.h:8,
                         from /home/user/ev3dev-kernel/kernel/bounds.c:9:
        /home/user/ev3dev-kernel/include/asm-generic/int-ll64.h:11:29: fatal error: asm/bitsperlong.h: No such file or directory
        compilation terminated.
        make[2]: *** [kernel/bounds.s] Error 1
        make[1]: *** [prepare0] Error 2
        make: *** [sub-make] Error 2

    Then you need to clean your kernel source tree like this:

         user@host ~/ev3dev-kernel $ git clean -dfX


[brickstrap]: https://github.com/ev3dev/brickstrap
[wiki]: https://github.com/ev3dev/ev3dev/wiki
