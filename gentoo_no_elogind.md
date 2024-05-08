# Running gentoo without elogind

For the past few months I've been running a gentoo machine with a focus on keeping things
lean whenever possible, and moving whatever packages and services into my VMs. The
profile I started with on the base system `default/linux/amd64/23.0/hardened`.

I don't have a gdm installed(just login from tty), just the X server configured,
and two window managers(xfce and xmonad). Initially the X server was built with
the `elogind` USE flag. My initial understanding was that running X server as
rootless required `elogind`. Turns out this is not true.

The first step is to remove `elogind` from the USE flags and recompile the X server and
drivers. Next, we make change the owning group for `Xorg` to `input`, and make it
a `setgid` binary. We also chown one of the `tty` as the user we want to login as.

```bash
$ chown -v :input /usr/bin/Xorg
$ chmod -v g+s /usr/bin/Xorg
$ chown -v eukabuka /dev/tty3
$ (tty3) startx ~/.xinitrc xmonad --vt3
```

Finally, remove elogind from the openrc boot level.

### References
* [How to Run Xorg as Non-root Without logind](https://www.youtube.com/watch?v=jT4fe-7Wmvs)
* [X server gentoo](https://wiki.gentoo.org/wiki/X_server)
* [Xorg guide gentoo](https://wiki.gentoo.org/wiki/Xorg/Guide)
