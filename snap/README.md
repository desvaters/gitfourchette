This folder contains boilerplate to produce a [Snap](https://snapcraft.io) package.

Snapcraft looks for its recipe in `snap/`, `build-aux/snap/` or the project root,
so unlike the other packaging boilerplate this doesn't live in `pkg/`.

## Building

Requires snapcraft and LXD:

```
sudo snap install snapcraft --classic
sudo snap install lxd
sudo lxd init --auto
sudo usermod -aG lxd "$USER"   # log out and back in afterwards
```

Then, from the repo's root directory:

```
snapcraft
```

This produces a `.snap` file in the repo's root directory. Install it with:

```
sudo snap install ./gitfourchette_*.snap --dangerous --classic
```

`--dangerous` merely acknowledges that the package isn't signed by the store,
and `--classic` acknowledges the confinement the recipe asks for.

There are no interfaces to connect afterwards: a classic snap runs outside the
sandbox and doesn't plug any.

## Why classic confinement, and why core26

These two choices are entangled, and each was forced by the one before it.

**Classic confinement** is upstream's call. GitFourchette drives host tooling
that a sandbox cannot reach: user-configured git hooks that call host programs,
external editors, terminals, diff and merge tools, credential helpers, and FUSE
mounts. There is no snap equivalent of Flatpak's `flatpak-spawn --host`. The
"Cost of strict confinement" section below records what that alternative
actually broke, measured.

**No Qt from a content snap** follows from classic. Snapcraft refuses to combine
a desktop extension with classic confinement -- `Extension 'kde-neon-6' does not
support confinement 'classic'`, same for `gnome`, verified with
`snapcraft expand-extensions` on 9.0.1. So no `kf6-core24`, and Qt has to be
staged from the distribution.

**core26** follows from that, because core24 cannot supply the Qt we need.
Ubuntu noble is Plasma 5: `kde-style-breeze` there depends on
`libqt5core5t64`/`libqt5widgets5t64`, `qt6-style-breeze` doesn't exist, and
there is no Qt6 Breeze style in noble at all. Ubuntu 26.04 is Plasma 6 and does
have it: `kde-style-breeze` 4:6.6.5 depends on `libqt6core6t64 (>= 6.10.2)`, and
`python3-pyqt6` is 6.10.2 -- bindings, Qt libraries and style plugin all come
from one release, so their ABI matches by construction. core26 ships no Python
in the base, but classic confinement forbids using the base's interpreter
anyway, so that cost is already paid.

## Why PyQt6 comes from the distribution

Upstream asks for two things at once: classic confinement, and an app that
follows the system-wide Qt style. The second one rules out the PyPI wheels.

The wheels bundle their own Qt build, which knows nothing about Breeze -- in
Kubuntu, an earlier wheel-based build offered only the Fusion and Windows
styles. Distribution packages give bindings, Qt libraries and style plugin from
a single release instead.

Third-party content snaps were considered and rejected:

- There is no PyQt6 runtime content snap for core24 or core26.
  `pyqt6-runtime-core24` doesn't exist; `qt6-core24` ships Qt6 without any PyQt
  bindings; KDE only publishes `kde-pyside6-core24-sdk`, which is build-time
  only.
- Content interfaces only auto-connect between snaps from the same publisher.
  Consuming someone else's runtime would need a snap declaration from Canonical
  for cross-publisher auto-connection.
- Content providers make no compatibility promises to consumers from other
  publishers.

## The Qt plugin blocker, and what it actually was

For a long stretch this recipe produced a snap that packed, installed and
staged everything, and then died at Qt initialisation:

    qt.qpa.plugin: Could not find the Qt platform plugin "wayland" in ""
    qt.qpa.plugin: Could not find the Qt platform plugin "xcb" in ""

**The empty `""` is a red herring.** It is not the plugin search path -- it is
`QT_QPA_PLATFORM_PLUGIN_PATH`, which is unset here and normally is. Running with
`QT_DEBUG_PLUGINS=1` shows Qt searching exactly the right directory and
rejecting every file it finds:

    checking directory path ".../qt6/plugins/platforms" ...
    looking at "libqxcb.so"
      ... 'libqxcb.so' is not a Qt plugin (metadata not found)

**The cause is patchelf 0.9, which snapcraft 9.0.1 bundles.** Qt 6.2 and later
keep a plugin's metadata in an ELF note and find it by walking the PT_NOTE
program headers. When patchelf 0.9 grows `.dynstr` to make room for an RPATH it
relocates `.note.qt.metadata` to the end of the file and does not extend PT_NOTE
coverage to the new location, so the note survives in the file but sits in a
LOAD segment where Qt never looks. Compare a staged plugin with the one from the
`.deb` it came from:

    pristine   05     .note.qt.metadata          <- a PT_NOTE segment
    staged     13     .dynamic .dynstr .note.qt.metadata   <- a LOAD segment

patchelf 0.18 from the distribution handles this correctly, with every flag
combination snapcraft uses (`--no-default-lib --force-rpath --set-rpath`). So it
is the version, not the invocation. The damage is also not repairable after the
fact: running 0.18 over an already-mangled plugin does not restore the coverage.

**Both halves of the fix are needed**, verified separately inside the installed
snap:

- Plugins that snapcraft's patchelf never touched are recognised again -- Qt
  reports "Available platform plugins are: xcb, vkkhrdisplay, offscreen,
  minimal, linuxfb, vnc" instead of finding nothing.
- But they then fail to *load*, because without an RPATH they cannot find
  `libQt6XcbQpa.so.6` and friends, which exist only inside the snap.

Hence the `qt-plugins` part: it stages the plugin tree out of snapcraft's
patchelf pass and sets the RPATH itself using patchelf 0.18. With a plugin
prepared that way, and no `LD_LIBRARY_PATH` anywhere, the app starts and the
staged `libqxcb.so` shows up in `/proc/<pid>/maps`.

This is worth reporting to Canonical: patchelf 0.9 predates Qt 6's note-based
plugin metadata by five years, and any snap staging Qt 6 plugins from a
distribution hits it.

## Fixed along the way

- **Entry point.** `$SNAP/usr/bin` came first on `PATH`, so the shebang's
  `env python3` found the staged distribution interpreter rather than the
  venv's, and the app couldn't import its own package.
- **QtSvg.** The distribution splits PyQt6 up per Qt module, so
  `python3-pyqt6.qtsvg` needs staging separately.
- **Library resolution.** Adding `override-prime` to the `qt` part silently
  suppressed snapcraft's patchelf pass, and every staged Qt library stopped
  resolving. `qt.conf` is generated from a separate part for that reason.
- **venv creation.** Once the `python` part is staged, its interpreter shadows
  the build environment's on `PATH`, so it needs `python3-venv` staged too.
  `PARTS_PYTHON_INTERPRETER` can't work around this: the plugin concatenates it
  onto a directory, so it has to stay a bare name.

## Verified working in the classic build

- The app starts and is usable. It picks the native Wayland platform plugin
  rather than falling back to Xwayland, and `breeze6.so` is loaded, so it
  follows the system-wide Qt style -- the second of upstream's two asks.
- `HOME` is the real home. Classic doesn't redirect it, so the override the
  strict recipe needed is gone, along with the `personal-files` paperwork.
- Host binaries are reachable. `/usr/bin/gh` resolves, so the credential helper
  limitation of the strict build is solved. Confirmed end to end: a fetch
  against a private repository succeeds without prompting, i.e. the host
  credential helper is actually used, not just present.
- The bundled git (2.53.0) wins on `PATH` and reads the real `~/.gitconfig`.

## Cost of strict confinement

This recipe used to be core24 + `confinement: strict`. It is kept here as the
measured case for classic, not as a description of the current build -- none of
it applies to the recipe in this folder any more.

Under strict confinement, a snap has no way to run a program on the host. That
affected external editors, terminals, diff and merge tools, and the user's own
`git` installation.

Git **credential helpers are host binaries** and were therefore unavailable. A
`~/.gitconfig` pointing `credential.helper` at the GitHub CLI failed inside the
snap with `/usr/bin/gh: not found`, and the same went for
`git-credential-manager`, `libsecret` and friends. It degraded rather than
hard-failed -- git falls back to GitFourchette's askpass, which does work in the
sandbox -- but it meant entering a token on every HTTPS operation.

The `home` interface only grants access to files that aren't hidden at the top
level of `$HOME`, so the recipe needed a `personal-files` plug for
`~/.gitconfig` and `~/.config/git`. `personal-files` is a super-privileged
interface: publishing to the Snap Store with it requires a manual review by
Canonical.

snapd points `$HOME` at the snap's private `$SNAP_USER_DATA`, so git and ssh
looked for `~/.gitconfig`, `~/.ssh/config` and `~/.ssh/known_hosts` in the wrong
place, and the recipe had to set `HOME` back to `$SNAP_REAL_HOME`.
GitFourchette's own preferences were unaffected, as they follow
`XDG_CONFIG_HOME`, which snapd keeps pointing inside the snap.

The host's **ssh-agent was out of reach**: its socket isn't covered by any
interface, and ssh inside the snap failed with
`ssh_get_authentication_socket: Permission denied`. Repositories using SSH
remotes therefore needed GitFourchette's own ssh-agent (the `ownSshAgent`
preference). That path did work: `ssh-agent` and `ssh-add` come with
`openssh-client`, the agent starts, its socket lands in the snap's private
`/tmp`, and the keys in `~/.ssh` are readable through the `ssh-keys` interface.

**This turned up an app bug that is worth fixing regardless of packaging:** both
of the app's checks for a system ssh-agent report a false positive inside a
strict snap. `SSH_AUTH_SOCK` is set (to the session agent's socket), and the
socket even passes `os.path.exists()` -- only `connect()` fails, with
`PermissionError`. So `trtables.py` won't append "(not detected)", and
`askpassdialog.py` believes an agent is available. With `ownSshAgent` defaulting
to false, a fresh install silently picks an agent it can't talk to. A reliable
probe has to actually connect to the socket.

Without the `mount-observe` interface, GIO failed to read `/proc/self/mountinfo`
on startup and logged a warning. Nothing seemed to depend on it, but connecting
the interface silenced it -- verified by comparing runs with and without.

## Debugging

Get a shell inside the snap's environment:

```
snap run --shell gitfourchette
```

Probes run there must use the venv interpreter at `$SNAP/bin/python3`, not the
staged distribution one at `$SNAP/usr/bin/python3` -- the latter knows nothing
about the venv, and using it produced a misleading diagnosis once already.

Ubuntu 26.04's rust-coreutils `timeout` crashes with SIGABRT and files an apport
report on every GUI run, so start the app in the background and kill it by PID
instead.

Watch for AppArmor/seccomp denials while using the app:

```
sudo snap install snappy-debug && sudo snappy-debug
```
