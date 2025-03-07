![Alt text](/img/screenshot.png?raw=true "virtme-ng screenshot")

What is virtme-ng?
====================

virtme-ng is a tool that allows to easily and quickly recompile and test a
Linux kernel, starting from the source code.

It allows to recompile the kernel in few minutes (rather than hours), then the
kernel is automatically started in a virtualized environment that is an exact
copy-on-write copy of your live system, which means that any changes made to
the virtualized environment do not affect the host system.

In order to do this a minimal config is produced (with the bare minimum support
to test the kernel inside qemu), then the selected kernel is automatically
built and started inside qemu, using the filesystem of the host as a
copy-on-write snapshot.

This means that you can safely destroy the entire filesystem, crash the kernel,
etc. without affecting the host.

Kernels produced with virtme-ng are lacking lots of features, in order to
reduce the build time to the minimum and still provide you a usable kernel
capable of running your tests and experiments.

virtme-ng is based on virtme, written by Andy Lutomirski <luto@kernel.org>
([web][korg-web] | [git][korg-git]).

Quick start
===========

```
 $ uname -r
 5.19.0-23-generic
 $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
 $ cd linux
 $ vng --build --commit v6.2-rc4
 ...
 $ vng --run
           _      _
    __   _(_)_ __| |_ _ __ ___   ___       _ __   __ _
    \ \ / / |  __| __|  _   _ \ / _ \_____|  _ \ / _  |
     \ V /| | |  | |_| | | | | |  __/_____| | | | (_| |
      \_/ |_|_|   \__|_| |_| |_|\___|     |_| |_|\__  |
                                                 |___/
    kernel version: 6.2.0-rc4-virtme x86_64

 $ uname -r
 6.2.0-rc4-virtme
 ^
 |___ Now you have a shell inside a virtualized copy of your entire system,
      that is running the new kernel! \o/

 Then simply type "exit" to return back to the real system.
```

Installation
============

You can clone this git repository and build a standalone virtme-ng running the
following commands:
```
 $ git clone --recurse-submodules https://github.com/arighi/virtme-ng.git
 $ BUILD_VIRTIOFSD=1 BUILD_VIRTME_NG_INIT=1 pip3 install --verbose -r requirements.txt .
```

Alternatively, if you're using Ubuntu, you can install virtme-ng from
ppa:arighi/virtme-ng:
```
 $ sudo add-apt-repository ppa:arighi/virtme-ng
 $ sudo apt install --yes virtme-ng
```

Requirements
============

 * You need Python 3.8 or higher

 * QEMU 1.6 or higher is recommended (QEMU 1.4 and 1.5 are partially supported
   using a rather ugly kludge)
   * You will have a much better experience if KVM is enabled.  That means that
     you should be on bare metal with hardware virtualization (VT-x or SVM)
     enabled or in a VM that supports nested virtualization.  On some Linux
     distributions, you may need to be a member of the "kvm" group.  Using
     VirtualBox or most VPS providers will fall back to emulation.

 * Depending on the options you use, you may need a statically linked `busybox`
   binary somewhere in your path.

Examples
========

 - Build and run tag v6.1-rc3 from a local kernel git repository:
```
   $ vng -b -c v6.1-rc3
```

 - Build and run a kernel 2 commits before the previously compiled kernel:
```
   $ vng -b -c HEAD~2
```

 - Run a kernel previously compiled from a local git repository:
```
   $ vng -r ./arch/x86/boot/bzImage
```

 - Test the kernel recompiled in the current working directory:
```
   $ vng -r
```

 - Test installed kernel 6.2.0-21-generic kernel
   (NOTE: /boot/vmlinuz-6.2.0-21-generic needs to be accessible):
```
   $ vng -r 6.2.0-21-generic
```

 - Download and test kernel 6.2.0-1003-lowlatency from deb packages:
```
   $ mkdir test
   $ cd test
   $ apt download linux-image-6.2.0-1003-lowlatency linux-modules-6.2.0-1003-lowlatency
   $ for d in *.deb; do dpkg -x $d .; done
   $ vng -r ./boot/vmlinuz-6.2.0-1003-lowlatency
```

 - Only generate .config in the current kernel build directory:
```
   $ vng --kconfig
```

 - Test the tip of the latest kernel, building the kernel on a remote build
   host called "builder", running make inside a specific build chroot
   (managed remotely by schroot):
```
   $ vng --build --build-host builder \
     --build-host-exec-prefix "schroot -c chroot:kinetic-amd64 -- "
```

 - Run the previously compiled kernel from the current working directory and
   enable networking:
```
   $ vng -r . --network user
```

 - Run the previously compiled kernel adding an additional virtio-scsi device:
```
   $ qemu-img create -f qcow2 /tmp/disk.img 8G
   $ vng --disk /tmp/disk.img
```

 - Recompile the kernel passing some env variables to enable Rust support
   (using specific versions of the Rust toolchain binaries):
```
   $ vng --build RUSTC=rustc-1.62 BINDGEN=bindgen-0.56 RUSTFMT=rustfmt-1.62
```

 - Build the arm64 kernel (using a separate chroot in /opt/chroot/arm64 as the
   main filesystem):
```
   $ vng --build --arch arm64 --root /opt/chroot/arm64/
```

 - Execute `uname -r` inside a kernel recompiled in the current directory and
   send the output to cowsay on the host:
```
   $ vng --exec 'uname -r' | cowsay
    __________________
   < 6.1.0-rc6-virtme >
    ------------------
           \   ^__^
            \  (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||
```

 - Run a bunch of parallel virtme-ng instances in a pipeline, with different
   kernels installed in the system, passing each other their stdout/stdin and
   return all the generated output back to the host (also measure the total
   elapsed time):
```
   $ time true | \
   > vng -r 5.19.0-38-generic -e "cat && uname -r" | \
   > vng -r 6.2.0-19-generic  -e "cat && uname -r" | \
   > vng -r 6.2.0-20-generic  -e "cat && uname -r" | \
   > vng -r 6.3.0-2-generic   -e "cat && uname -r" | \
   > cowsay -n
    ___________________
   / 5.19.0-38-generic \
   | 6.2.0-19-generic  |
   | 6.2.0-20-generic  |
   \ 6.3.0-2-generic   /
    -------------------
           \   ^__^
            \  (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||

   real    0m2.737s
   user    0m8.425s
   sys     0m8.806s
```

 - Run `glxgears` inside a kernel recompiled in the current directory:
```
   $ vng -g glxgears

   (virtme-ng is started in graphical mode)
```

 - Execute an `awesome` window manager session with kernel
   6.2.0-1003-lowlatency (installed in the system):
```
   $ vng -r 6.2.0-1003-lowlatency -g awesome

   (virtme-ng is started in graphical mode)
```

 - Run the `steam` snap (tested in Ubuntu) inside a virtme-ng instance using
   the 6.2.0-1003-lowlatency kernel:
```
   $ vng -r 6.2.0-1003-lowlatency --snaps --net user -g /snap/bin/steam

   (virtme-ng is started in graphical mode)
```

Implementation details
======================

virtme-ng allows to automatically configure, build and run kernels using the
main command-line interface called `vng`.

A minimal custom `.config` is automatically generated if not already present
when --build is specified.

It is possible to specify a set of custom configs (.config chunk) in
`~/.config/virtme-ng/kernel.config`, these user-specific settings will override
the default settings (except for the mandatory configs that are required to
boot and test the kernel inside qemu, using `virtme-run`).

Then the kernel is compiled either locally or on an external build host (if the
`--build-host` option is used); once the build is done only the required files
needed to test the kernel are copied from the remote host if an external build
host is used.

When a remote build host is used (`--build-host`) the target branch is force
pushed to the remote host inside the `~/.virtme folder`.

Then the kernel is executed using the virtme module. This allows to test the
kernel using a safe copy-on-write snapshot of the entire host filesystem.

All the kernels compiled with virtme-ng have a `-virtme` suffix to their kernel
version, this allows to easily determine if you're inside a virtme-ng kernel or
if you're using the real host kernel (simply by checking `uname -r`).

External kernel modules
=======================

It is possible to recompile and test out-of-tree kernel modules inside the
virtme-ng kernel, simply by building them against the local directory of the
kernel git repository that was used to build and run the kernel.

Default options
===============

Typically, if you always use virtme-ng with an external build server (e.g.,
`vng --build --build-host REMOTE_SERVER --build-host-exec-prefix CMD`) you
don't always want to specify these options, so instead, you can simply define
them in `~/.config/virtme-ng/virtme-ng.conf` under `default_opts` and then
simply run `vng --build`.

Example (always use an external build server called 'kathleen' and run make
inside a build chroot called `chroot:lunar-amd64`). To do so, modify the
`default_opts` sections in `~/.config/virtme-ng/virtme-ng.conf` as following:
```
    "default_opts" : {
        "build_host": "kathleen",
        "build_host_exec_prefix": "schroot -c chroot:lunar-amd64 --"
    },
```

Now you can simply run `vng --build` to build your kernel from the current
working directory using the external build host, prepending the exec prefix
command when running make.

Troubleshooting
===============

 - If you get permission denied when starting qemu, make sure that your
   username is assigned to the group `kvm` or `libvirt`:
```
  $ groups | grep "kvm\|libvirt"
```

 - When using `--network bridge` to create a bridged network in the guest you
   may get the following error:
```
  ...
  failed to create tun device: Operation not permitted
```

   This is because `qemu-bridge-helper` requires `CAP_NET_ADMIN` permissions.

   To fix this you need to add `allow all` to `/etc/qemu/bridge.conf` and set
   the `CAP_NET_ADMIN` capability to `qemu-bridge-helper`, as following:
```
  $ sudo filecap /usr/lib/qemu/qemu-bridge-helper net_admin
```

 - If the guest fails to start because the host doesn't have enough memory
   available you can specify a different amount of memory using `--memory MB`,
   (this option is passed directly to qemu via `-m`, default is 1G).

 - If you're testing a kernel for an architecture different than the host, keep
   in mind that you need to use also `--root DIR` to use a specific chroot with
   the binaries compatible with the architecture that you're testing.

   If the chroot doesn't exist in your system virtme-ng will automatically
   create it using the latest daily build Ubuntu cloud image:
```
  $ vng --build --arch riscv64 --root ./tmproot
```

 - If the build on a remote build host is failing unexpectedly you may want to
   try cleaning up the remote git repository, running:
```
  $ vng --clean --build-host HOSTNAME
```

 - Snap support is still experimental and something may not work as expected
   (keep in mind that virtme-ng will try to run snapd in a bare minimum system
   environment without systemd), if some snaps are not running try to disable
   apparmor, adding `--append="apparmor=0"` to the virtme-ng command line.

Contributing
============

Please see DCO-1.1.txt.

Credits
=======

virtme-ng is written by Andrea Righi <andrea.righi@canonical.com>

virtme-ng is based on virtme, written by Andy Lutomirski <luto@kernel.org>
([web][korg-web] | [git][korg-git]).

[korg-web]: https://git.kernel.org/cgit/utils/kernel/virtme/virtme.git "virtme on kernel.org"
[korg-git]: git://git.kernel.org/pub/scm/utils/kernel/virtme/virtme.git "git address"
[virtme]: https://github.com/amluto/virtme "virtme"
[virtme-ng-ppa]: https://launchpad.net/~arighi/+archive/ubuntu/virtme-ng "virtme-ng ppa"
