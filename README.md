# Container-Notes
Notes about how to get full distros to run in different container systems

## Who this document is for
This document is for those who already know how to use chroot, systemd-nspawn, lxd, docker and whatnot. It is for those that got to the point of wanting to use full fledged system containers but wondering why it's so difficult to get the distro you happen to want to use into a container of your choice and if you managed to do so why they still won't let you log in sometimes or even fail to start their init system in the first place on some occassions.  
For advice on how to get distros into containers refer to the last part of https://www.freedesktop.org/software/systemd/man/systemd-nspawn.html  
This basically applies to lxd or docker the same way as it's always just a rootfs that is chrooted into. Often one can even just copy the files from a live-ISO and it would work.
Case in point: https://tutorials.ubuntu.com/tutorial/create-custom-lxd-images#0

## Introduction
All container systems are basically super-chroots. This means that there always has to be an entrypoint from which to execute the programm, shell or init-system. Often there will be an /sbin/init symlink pointing to the distros init-systme, be that systemd, openrc or something else.  
So far so good. 


_Does this mean I just need to do it like I do it in docker and write a DOCKERFILE with an ENTRYPOINT-line?_

![Well yes](https://en.meming.world/images/en/thumb/f/f7/Well_Yes%2C_But_Actually_No.jpg/300px-Well_Yes%2C_But_Actually_No.jpg)

If you do a `chroot` or a `systemd-nspawn` you usually do it like this `chroot /path/to/new/root /path/to/thing2run/in/new/root`  
This is where things start to get more complicated than just using a docker container. 
Systems like docker, rkt or containerd are pure application-container-runtimes. They allow for interactivity via attaching to the containers stdin,stderr,stdout but are not designed for attaching to virtual-consoles (like /dev/console) inside of the container.

## Enter `systemd-nspawn` and the magic `-b`-flag
What does the `-b` flag do?  
Basically it just tries to boot systemd inside a container and does lots of setup magic.  
_So it's useless for openRC?_  
Well, no but it might not work as flawlessly with it, the `-b` flag actually just starts `/sbin/init`.  
If that points to systemd it and sbus will take care of virtual consoles and /dev/console.  
Still this does not mean openRC won't work just as well. You just might have to adjust `/etc/initab` and or `/etc/securetty` [1]

If the problem is that you cannot log in due to the console being spammed by something along the lines of `cannot attach tty0` you just comment out all ttyN lines in `/etc/initab` and make sure that there is one `/dev/console` line as systemd-nspawn uses that one to attach interactively.


NOTE: If you also use `machinectl` its `login` command varies from that as it uses the `pty/0` device which needs to be present in `/etc/securetty` if that file exists.

_Great! Now I see my login prompt. Why won't it log in with root and no password?_  
systemd-nspawn and machinectl both act as root if you use them in shell mode, which is basically the equivalent to `chroot /newroot /bin/bash` if you use them in system container mode though they require password for root. So just chroot into them or use shell mode and set a root password via `passwd root`

_Okay, great this works for distro X but not for distro Y_  
In this case don't use the `-b` flag but do it the chroot way and point systemd-nspawn to your init-systems executable. Possibly even use `--as-pid2` which would create a stub-process as PID1 which immediately hands off to the pointed to executable.

## Summary
Even with init-systems container stuff is still more or less chrooting to an entry-executable.
init-systems start login-daemons though, which might use authentication-modules like PAM-modules and also start virtual console devices to log in on.  
Problems with login in system containers can originate from either trying to start a console device not supported by the container-runtime. systemd-nspawn uses /dev/console here while machinectl uses /dev/pty/0-N both require passwords for root being set. Both can usually be adjusted via oldschool chroot and editing `/etc/initab`, `/etc/securetty` and doing a quick `passwd root`

_Wait, wait, wait, wait. What if I want to use super exotic distros like GuixSD?_  
Surprisingly even with Distros like GuixSD that forgoe the standard filesystem-hierarchy it's mostly the exact same process.  
Though the above needs to be done differently in Guix and there are additional steps required.

First things first though. How to get a guix container image? You use the `guix system reconfigure [guix-system.scm file]` command in a VM and copy the files.  
_Wait what? Why in a VM though. guix runs everywhere!_  
True but for the `guix system` commands that actually put out a filesystem to succeed _and_ actually finish, the mandatory filesystem and bootloader expressions in the system.scm may not fail.  
In addition to that you can't use the (implicitly) default %base-services as those setup mintty with all those pesky tty-devices instead of /dev/console.

the service part of the file should therefore look like this:
```scheme
(services
         (list
           (udev-service) ;;required for tty console
           (guix-service)
           (login-service);;required for obvious reasons. root pswd is empty btw.
           (agetty-service
             (agetty-configuration
               (term "linux")  ;;makes terminal colors work.
               (tty "console") ;;because no access to /dev/tty
             )
           )
```
For use with machinectl and/or lxd, `(dbus-service)` might be required too as those start their pty/0 via dbus.  
Now that we have a GuixSD rootfs we need to add two things before systemd-nspawn will run it. We need to create a `/etc/os-release` file and create the `/usr` directory. Otherwise systemd-nspawn will complain about not finding a valid OS-tree.

After all this is done GuixSD can be booted with a command of this form: `systemd-nspawn -D /GuixSD-rootfs '/gnu/store/longass-hash-shepherd/bin/shepherd' '--config' '/gnu/store/longass-hash-shepherd.conf'`  
This will get you all the way to the login prompt and allow login. At the time of this writing it will fail to load the shell's `profile` or `bashrc` though. This can be done manually but is a real pain in the ass due to those long hashes in /gnu/store. Also the user-specific bashrc which would point to the users guix-profile's bin's paths is missing.

---
[1] https://github.com/systemd/systemd/issues/685
