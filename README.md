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

# When to use `systemd`  vs `systemdMinimal` vs `systemdLibs`



# How can we make `systemd.pc` do the right thing?

Currently `systemd.pc` instructs packages to install systemd units etc into systemd's nix store path
which obviously doesn't work. Find a smart way to make this work?


`util-linux -> lvm2 `
