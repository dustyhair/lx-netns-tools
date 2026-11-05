# lx-netns-tools

`lx-netns-tools` connects stock bubblewrap to the private VNIC pool assigned
by the SmartOS LX brand. It contains no global-zone controller and does not
create host links, forwarding rules, or NAT rules.

Configure private networking while the zone is stopped:

```sh
vmadm update UUID \
    lx_private_network=true \
    lx_private_network_egress=e1000g0 \
    lx_private_network_subnet=10.88.0.0/24 \
    lx_private_network_links=2
vmadm start UUID
```

The brand writes a host-owned configuration file and exposes it read-only at
`/native/.zonecontrol/lx-private-network.conf`. Install this package and the
distribution's normal `bubblewrap` package, then call `lx-bwrap` with ordinary
bubblewrap arguments:

```sh
lx-bwrap \
    --unshare-user \
    --ro-bind / / \
    -- /usr/bin/curl -I https://example.com/
```

Use a full-root bind on SmartOS LX.  Partial-root layouts such as separate
`/usr`, `/etc`, `/dev`, and `/proc` mounts do not yet reconstruct a usable
root in an LX mount namespace.  Add narrower writable mounts after the
full-root bind when the payload needs them.

`lx-bwrap` adds `--unshare-net` if needed. The helper claims the first
available assigned VNIC, brings it up, adds its IPv4 address and default
route, waits for duplicate-address detection, and drops `CAP_NET_ADMIN` and
`CAP_SETPCAP` from every capability set before it executes the command.
Networked `lx-bwrap` invocations are serialized for their complete lifetime,
including a one-second teardown grace period. The launcher keeps its lock file
descriptor out of the payload, forwards HUP, INT, and TERM to bubblewrap, and
retains the lock through the grace period. Before it executes bubblewrap, the
wrapper arms a Linux parent-death signal. It also forces bubblewrap's
`--die-with-parent` behavior. Killing the launcher therefore kills bubblewrap
and the payload. `SIGKILL` still bypasses the one-second grace period, so the
next launch may wait briefly for host VNIC teardown.

Use a SmartOS platform that includes the LX detached-network-stack concurrency
fixes. Older platforms can panic if one VNIC-backed network namespace starts
while another is being destroyed. Guest serialization reduces that risk but is
not a host safety boundary because a guest can bypass `lx-bwrap`.

Version 0.1.4 allows one active networked sandbox per LX zone. Additional links
support other namespace managers and future releases.

Stop all `lx-bwrap` processes before installing or upgrading the package.
Version 0.1.4 locks the package directory so later package upgrades keep the
same lock inode, but an upgrade from 0.1.2 changes the lock location.

The argument rewriter follows the bubblewrap 0.11.2 option table. It rejects
unknown options rather than risk interpreting an option operand as the
command separator. `--args` is unsupported because it can hide options in a
file descriptor, and `--share-net` conflicts with this command's purpose.

The package currently supports x86-64 LX guests. `make test` builds both
helpers and checks version consistency, argument rewriting, lock descriptor
ownership, serialized launches, signal forwarding, exit status, and teardown
grace. Build the package formats with:

```sh
make deb
make xbps
```

The package depends on `util-linux` for `flock`. The `deb` target requires
`dpkg-deb`, and the `xbps` target requires `xbps-create`. The helper is linked
statically with the libc selected by `CC`; building on Void does not by itself
select musl. The libc and target architecture policy remains packaging work
for a future release.

## Host setup

The SmartOS global-zone root is a ramdisk. Editing `/etc/system` or persistent
`ipadm` properties in the running global zone is not enough to survive a
reboot.

For a ZFS-booted standalone host, run the supplied installer from the release
bundle or source tree:

```sh
pfexec smartos/install-lx-private-network-host install
```

The installer preserves existing local configuration, adds each snippet once,
and stops if it finds a partial installation. It keeps the `/etc/system`
override under the boot pool, updates the active `loader.conf.local`, installs
the SMF method and manifest under `/opt/custom`, imports the service, and
enables it. Repeating the command is safe.

The system snippet enables these six experimental gates:

```text
set lx_brand:lx_seccomp_experimental=1
set lx_brand:lx_userns_experimental=1
set lx_brand:lx_mntns_experimental=1
set lx_brand:lx_pidns_experimental=1
set lx_brand:lx_ipcns_experimental=1
set lx_brand:lx_netns_experimental=1
```

The SMF service enables IP Filter and IPv4 forwarding, verifies both, and
orders zone and `vmadmd` startup after those checks. `/zones/boot/boot`
resolves to the active boot image. Rerun the installer after selecting a
different boot image with `piadm`. Platform-only activation does not change
that link.

Reboot when the installer reports `reboot_required=1`, then verify the complete
host setup before starting an enabled zone:

```sh
pfexec smartos/install-lx-private-network-host check
```

The LX lifecycle hook checks these settings. It does not change global host
state. Each enabled VM needs a distinct RFC 1918 `/24`. The egress datalink
must exist in the global zone.

## Install and upgrade

Install the distribution's `bubblewrap` and `util-linux` packages first. Then
install the package built for the guest:

```sh
# Debian or Ubuntu
dpkg -i lx-netns-tools_0.1.4_amd64.deb

# Void Linux
xbps-install --repository=/path/to/packages lx-netns-tools
```

Stop every running `lx-bwrap` process before upgrading from 0.1.2. The lock
location changed in 0.1.4.

## Checks and recovery

Run `lx_private_network status` in the global zone to list active deployments.
If a VM stop leaves `cleanup_pending` state, retry the normal VM stop. If the VM
is already stopped, run the cleanup hook with the exact UUID and zone path:

```sh
/usr/lib/brand/lx/lx_private_network cleanup UUID /zones/UUID
```

Do not delete global IP Filter rules, NAT rules, datalinks, or files under
`/var/run/lx-private-network` by hand.

Private networking is experimental. It provides routed IPv4 and source NAT.
It does not provide IPv6, inbound port forwarding, per-namespace DNS policy,
bandwidth controls, or automatic subnet allocation. The current guest launcher
allows one networked sandbox at a time per VM.
