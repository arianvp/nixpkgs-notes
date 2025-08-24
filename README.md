# Circular dependencies

`tpm2-tss -> systemd -> tpm2-tss`

tpm2-tss ships with sysusers, tmpfiles and udev rules it wants to install
but only does that if we have a `systemd` dependency.

We don't have it depend on `systemd` atm so it falls back to using `useradd` and `userdel` but we patch that behaviour out.

We then do everything in a NixOS module instead


`tpm2-tss[test]-> procps -> systemd -> tpm2-tss`

The tests of tpm2-tss depend on procps. procps depends on systemd. systemd depends on tpm2-tss. This creates a loop.

Current fix:  `procps` overriden locally to set `withSystemd = false`

Suggested fix: Move tests into own derivation.

`util-linux -> systemd -> util-linux`
`systemd -> linux-pam -> systemd`
`systemd -> util-linux -> linux-pam -> systemd`

```
$ nix why-depends --derivation nixpkgs#util-linux nixpkgs#util-linuxMinimal
/nix/store/8vimrp4862ipgd2g3hi0d4j51jvpwkxc-util-linux-2.41.drv
└───/nix/store/2370y84xrb6mf46rjj63zr7ipcrkcyiz-systemd-257.6.drv
    └───/nix/store/mncaq3nimi056aqbwnkx5w0vw1gf1p1d-util-linux-minimal-2.41.drv

`util-linux` depends on `systemd` for reasons:

* To load `systemdsystemunitdir`, `sysuser_dir` and `tmpfiles_dir` from `systemd.pc`
  * `uuidd` installs `sysusers`, `tmpfiles` and `uuidd.{service,socket}`
    * to create the `uuidd` user
    * to create `@{runstatedir,localstatedir}/uuidd 2775 uuidd uidd`
      * Can not rely on the systemd unit file to create these dirs because OpenRC depends on `systemd-tmpfiles`
  * `lastlog2` installs `lastlog2-import.service` to migrate `/var/log/lastlog` to `/var/log/lastlog/lastlog2.db`
  * `fstrim` installs `fstrim.{service,timer` to trim filesystems

* For `libsystemd`
  * `lslogins` depends on `libsystemd` for `<systemd/sd-journal.h>`
  * `zramctl` depends on `libsystemd` for `sd_device` to wait for zram to be initialized (NEW)
  * `agetty` depends on `libsystemd` for `<systemd/sd-daemon.h>` and `<systemd/sd-login.h>` it calls `sd_booted()` and `sd_session`. to replace `utmp`
  * `term-utils/wall` depends on `libsystemd` for ? `<systemd/sd-login.h>` and `<systemd/sd-daemon.h>`. it calls `sd_booted()` and `sd_session*` to replace `utmp`
  * `term-utils/write` depends on `libsystemd` for `<systemd/sd-login.h>` and `<systemd/sd-daemon.h>`. it calls `sd_booted()` and `sd_session*` to replace `utmp`
  * `logger` depends on `libsystemd` for `<systemd/sd-daemon.h>`  and `<systemd/sd-journal.h>`
  * `uuidd` depends on `libsystemd` for `<systemd/sd-daemon.h>`

Good news is that these are all **programs** and not **libraries**


`systemd` depends on `util-linux` for the following  libraries:

* `liblkid`
* `libfdisk`
* `libmount`
* `(libuuid)` ?

`systemd` depends on `util-linux` for the following binaries during runtime: `umount mount swapon swapoff sugloin agetty fsck`


Current fix:
  * `systemd` is built with `utilLinuxMinimal`
  * runtime dependencies point to `utilLinuxMinimal.{login,mount,swap}`
  * All the `getty` services shipped with systemd simply are broken and point to `/sbin/agetty` and are fixed in the NixOS module system instead :(

`nologin` and `sulogin` both do not depend on `linux-pam` or `libystemd`

We did the split outputs to "minimize" the closure of systemd. But there
is little point in my opinion given that we still pull in the *full* util-linux later for `agetty` ?

For `systemdLibs` we do not include runtime references to `util-linux` anyway
so it's already minimal.

(Future) problems:

* `agetty` links against systemd `libsystemd` so we need to really point it to the
  "runtime" version of agetty which is linked against us. That currently
  happens due to the NixOS module and the agetty services shipped with systemd package being 'broken' but https://github.com/systemd/systemd/pull/38696 will unbreak them and will cause us to point the agetty service to an agetty without
  logind support which is not good.

Sidenote about shadow/util-linux weirdness:

* We point our `agetty` to `${pkgs.shadow}/bin/login` as opposed to
  the default `${pkgs.util-linux}/bin/login`  which `agetty` will
  use by default.
  We are a weird exception compared to other distros AFAIK.
*  We use ${shadow}/bin/su whilst other distros use ${utilLinux}/bin/su
*  We ship `${shadow}/bin/nologin` in /run/current-system/sw/bin even-though we configure systemd to use `${util-linux}/bin/nologin`


Future fix:
* We should probably point all these paths to `/run/current-system/sw/bin` so
  they point to the "runtime" version of `util-linux`.
  For example,


```
$ nix why-depends --derivation nixpkgs#systemd  nixpkgs#systemdLibs --all
/nix/store/2370y84xrb6mf46rjj63zr7ipcrkcyiz-systemd-257.6.drv
├───/nix/store/a5vbg7z1ci0xnlc6r9b1d9pwznyxijn8-libfido2-1.16.0.drv
│   ├───/nix/store/kya8fwm8abcrcfdhwfgc7wq8viibp6y7-systemd-minimal-libs-257.6.drv
│   └───/nix/store/gm64y20jypfi51j7asrvnw944cnbpprl-pcsclite-2.3.0.drv
│       └───/nix/store/kya8fwm8abcrcfdhwfgc7wq8viibp6y7-systemd-minimal-libs-257.6.drv
└───/nix/store/pjafn3dksvs6jgx0pkb6jcn5p28w78d0-cryptsetup-2.8.0.drv
    └───/nix/store/yymh1rn3vc268gjimy43g8ldpaiyrj6p-lvm2-2.03.33.drv
        └───/nix/store/kya8fwm8abcrcfdhwfgc7wq8viibp6y7-systemd-minimal-libs-257.6.drv
```

```
$ nix why-depends --derivation nixpkgs#systemd  nixpkgs#systemdMinimal --all
/nix/store/2370y84xrb6mf46rjj63zr7ipcrkcyiz-systemd-257.6.drv
├───/nix/store/a5vbg7z1ci0xnlc6r9b1d9pwznyxijn8-libfido2-1.16.0.drv
│   ├───/nix/store/0xz72aqjmi62x86afbqzqrb4fdkpqlxw-udev-check-hook.drv
│   │   └───/nix/store/a3y0fmh0kilz9ynvk2m2zcrdrn4mcadp-systemd-minimal-257.6.drv
│   └───/nix/store/gm64y20jypfi51j7asrvnw944cnbpprl-pcsclite-2.3.0.drv
│       └───/nix/store/gxyb7nk6d2b17q3isr4zmmszq9dmly7c-dbus-1.14.10.drv
│           └───/nix/store/a3y0fmh0kilz9ynvk2m2zcrdrn4mcadp-systemd-minimal-257.6.drv
└───/nix/store/pjafn3dksvs6jgx0pkb6jcn5p28w78d0-cryptsetup-2.8.0.drv
    └───/nix/store/yymh1rn3vc268gjimy43g8ldpaiyrj6p-lvm2-2.03.33.drv
        └───/nix/store/0xz72aqjmi62x86afbqzqrb4fdkpqlxw-udev-check-hook.drv
```


# `systemd`  vs `systemdMinimal` vs `systemdLibs`

When to use which? Why does `systemdMinimal` exist? Audit usages?

* one thing that we need to be very careful with is that `sd_path` is a thing and as soon as anything
depends on it things might go wrong. It's basically the same problem as `systemd.pc`


# How can we make `systemd.pc` do the right thing?

Currently `systemd.pc` instructs packages to install systemd units etc into systemd's nix store path
which obviously doesn't work. Find a smart way to make this work?


# Things to do when 258 drops

* We can drop the explicit `gnutls` dependency. as `systemd-resolved` uses
  `openssl` now. `journal-remote` still depends on it through `libmicrohttpd`
  though.
