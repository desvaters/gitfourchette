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

## Debugging

Get a shell inside the snap's confined environment:

```
snap run --shell gitfourchette
```

Watch for AppArmor/seccomp denials while using the app:

```
sudo snap install snappy-debug && sudo snappy-debug
```
