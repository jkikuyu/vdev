vdev: a virtual device manager and filesystem
=============================================

**This system is a work-in-progress.  If you would like to help, please see the [Issue Tracker](https://github.com/jcnelson/vdev/issues)**.

vdev is a userspace device manager and filesystem that exposes attached devices as device files for UNIX-like operating systems.  It differs from FreeBSD's devfs or Linux's udev, eudev, and mdev in that it offers a filesystem interface that implements a *per-process view of /dev* in a portable manner, providing the necessary functionality for the host administrator to control device node visibility and access based on arbitrary criteria (e.g. process UID, process session, process seat, etc.).

More information is available in the [design document](http://judecnelson.blogspot.com/2015/01/introducing-vdev.html).

Project Goals
-------------
* **Portable Architecture**.  Unlike devfs or udev, vdev is designed to be portable across (modern) *nix.  As such, all OS-specific functionality is factored out of the core logic, and the core logic makes minimal assumptions about the underlying OS's device API.  In addition, the architecture supports multiple hardware event sources, including non-OS sources (such as another vdev instance, or modifications to an existing /dev tree).  As a consequence, vdev will make every reasonable effort to build with non-GNU libc's and support static linking.
* **Event-driven**.  Like devfs and udev, vdev leverages the underlying OS's hardware event-notification APIs.  Device nodes appear and disappear as the devices themselves become available or unavailable.  As with other device managers, the administrator can define actions for vdev to take when a device's state changes.
* **Advanced Access Control**.  Unlike existing systems, vdev comes with an optional userspace filesystem that lets the administrator control visibility and access to device files based on which process is asking (equivalent to giving each process its own view of /dev).  The administrator does so by defining ACLs that bind a set of paths to a predicate on the calling process's PID.  Vdev's add-on filesystem evaluates the predicate on each access to determine whether or not the the set of paths will be visible to the calling process, as well as which user/group/permissions the paths will appear to have.  This helps decouple device access control from orthogonal administrative tasks like session management and seat management.
* **Container-Friendly**.  Vdev runs without modification within containers and jails.  Like devfs, it exposes only the devices the root context wishes the contained process to see.  In addition, it is possible to run multiple instances of vdev concurrently, alongside devfs or udev.

Project Non-Goals
-----------------
* **Init system integration**.  Vdev is meant to run with full functionality regardless of which init system, daemon manager, service manager, plumbing layer, etc. the user runs, since they address orthogonal concerns.  It will coexist with and complement them, but neither forcibly replace them nor require them to be present.

Dependencies
-----------

There are two binaries in vdev:  the hotplug daemon vdevd, and the userspace filesystem vdevfs.  You can use one without the other.

To build vdevd, you'll need:
* libc
* libpthread

For vdevfs, you'll need all of the above, plus:
* libstdc++
* FUSE
* [libpstat](https://github.com/jcnelson/libpstat)
* [fskit](https://github.com/jcnelson/fskit)

Building
--------

To do things the fast way, with default options:

    $ make
    $ sudo make install 


To do things the careful way, you'll need to build and install libvdev.  To do so, use the following commands:

    $ make -C libvdev 
    $ sudo make -C libvdev install

To build and install vdevd, type:

    $ make -C vdevd OS=$OS_TYPE
    $ sudo make -C vdevd install

Substitute $OS_TYPE with:
* "LINUX" to build for Linux
* "FREEBSD" to build for FreeBSD (coming soon)
* "OPENBSD" to build for OpenBSD (coming soon)

$OS_TYPE defaults to "LINUX".

To build vdevfs, type:

    $ make -C fs
    $ sudo make -C fs install


By default, libvdev is installed to /lib/, vdevd is installed to /sbin/vdevd, vdevd's helper programs are installed to /lib/vdev/, and vdevfs is installed to /usr/sbin/vdevfs.  You can override any of these directory choices at build-time by setting the "PREFIX=" variable on the command-line (e.g. `make -C libvdev PREFIX=/usr/local/lib`), and you can override the installation location by setting "DESTDIR=" at install-time (e.g. `sudo make -C libvdev install DESTDIR=/usr/local/lib`).

**A Note on Static Linking**

You can directly set `CFLAGS` and `LDFLAGS` to build a statically-linked vdevd and a `libvdev.a` file, and you should be able to link against non-GNU libcs.  However, if you choose to do so, you should not run multiple jobs in parallel (e.g. running `make -j1` will guarantee this).  This is due to the way the archive file is generated; this will be improved at a later date.

FAQ
---
* **Why another device manager?**  I want to control which processes have access to which device nodes based on criteria other than their user and group IDs.  For example, I want to filter access based on which login session the process belongs to, and I want to filter based on which container a process runs in.  Also, I want to have this functionality available on each of the *nix OSs I use on a (semi-)regular basis.  To the best of my knowledge, no device manager lets me do this (except perhaps Plan 9's per-process namespaces), so I wrote my own.
* **What is the license for vdev?**  Vdev code is available for use under the terms of either the [GPLv3+](https://github.com/jcnelson/vdev/blob/master/LICENSE.GPLv3%2B) or [ISC license](https://github.com/jcnelson/vdev/blob/master/LICENSE.ISC).  However, the Linux port of vdev ships with some optional Linux-specific [helper programs](https://github.com/jcnelson/vdev/tree/master/vdevd/helpers/LINUX) that were derived from udev.  They are available under the terms of the GPLv2 where noted in the source files.
* **How much time do you have to spend on this project?**  This is a side-project that's not directly related to my day job, so nights and weekends when I have cycles to spare.  However, I am very interested in the success of this project, so expect frequent releases nevertheless :)
* **Which Linux distributions are you targeting?**  vdev is distro-agnostic, but I'm developing and testing primarily for [Debian](http://www.debian.org) and [Devuan](http://devuan.org).
* **Which BSDs/other UNIXes are you planning to support?**  OpenBSD and FreeBSD for now.  Well-written, suitably-licensed pull requests that port vdev to these or other UNIX-like OSs will be gratefully accepted.
* **Will vdev have a DBus API, or a C library?**  These are not necessary.  Vdev will expose its API through the filesystem directly.  However, it will also ship with a libudev compatibility library.
* **Does this project have anything to do with systemd and udev merging?**  It's not strictly related, but the pending merger definitely motivated me to start this project sooner rather than later.  I hope vdev's existence will help alleviate the tension between the pro- and anti-systemd crowds:
  * Linux users who don't want to use systemd can run vdev in place of udev.
  * Non-systemd and non-udev users who rely on software that depends on libudev can use libudev-compat.
  * Linux developers who need fine-grained device access controls but don't want to couple their software to systemd will have vdevfs as a portable, easy-to-use alternative.
  * Users do not need to choose between systemd and vdevfs, since they can coexist.
  * Systemd developers can go ahead and tightly-couple udev to systemd-PID-1.
  * **Result:** Everyone wins.
