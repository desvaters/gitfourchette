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
sudo snap install ./gitfourchette_*.snap --dangerous
```

`--dangerous` merely acknowledges that the package isn't signed by the store.

## Interfaces

Some of the interfaces this snap requests aren't connected automatically.
After installing, connect them manually:

```
sudo snap connect gitfourchette:ssh-keys
sudo snap connect gitfourchette:dot-git-config
sudo snap connect gitfourchette:password-manager-service
sudo snap connect gitfourchette:removable-media
```

## Known limitations

Under **strict confinement**, a snap has no way to run a program on the host —
there's no equivalent to Flatpak's `flatpak-spawn --host`. This affects:

- External editors, terminals, diff tools and merge tools.
- The user's own `git` installation. This snap bundles git instead, which is
  why the Flatpak's "built-in git" isn't optional here.

The `home` interface only grants access to files that aren't hidden at the top
level of `$HOME`, hence the `personal-files` plug for `~/.gitconfig` and
`~/.config/git`. `personal-files` is a super-privileged interface: publishing to
the Snap Store with it requires a manual review by Canonical.

snapd points `$HOME` at the snap's private `$SNAP_USER_DATA`, so git and ssh
would look for `~/.gitconfig`, `~/.ssh/config` and `~/.ssh/known_hosts` in the
wrong place. This recipe sets `HOME` back to `$SNAP_REAL_HOME`. GitFourchette's
own preferences are unaffected, as they follow `XDG_CONFIG_HOME`, which snapd
keeps pointing inside the snap.

The host's **ssh-agent is out of reach**: its socket isn't covered by any
interface, and ssh inside the snap fails with
`ssh_get_authentication_socket: Permission denied`. Repositories using SSH
remotes therefore need GitFourchette's own ssh-agent (the `ownSshAgent`
preference) rather than the session agent.

## Why PyQt6 comes from PyPI

The KDE neon extensions state outright that they "don't provide the bindings
needed for PySide2 (Qt for Python) or PyQt apps", so the Qt bindings have to
come from somewhere else. Third-party content snaps were considered and
rejected:

- There is no PyQt6 runtime content snap for core24. `pyqt6-runtime-core24`
  doesn't exist; `qt6-core24` ships Qt6 without any PyQt bindings; KDE only
  publishes `kde-pyside6-core24-sdk`, which is build-time only.
- Content interfaces only auto-connect between snaps from the same publisher.
  Consuming someone else's runtime would require yet another manual store
  review to get a snap declaration for cross-publisher auto-connection --
  on top of the one `personal-files` already needs.
- Content providers make no compatibility promises to consumers from other
  publishers.

The PyPI wheels cost roughly 95 MB before squashfs compression. That's the
price of not depending on a stranger's runtime.

## Debugging

Get a shell inside the snap's confined environment:

```
snap run --shell gitfourchette
```

Watch for AppArmor/seccomp denials while using the app:

```
sudo snap install snappy-debug && sudo snappy-debug
```
