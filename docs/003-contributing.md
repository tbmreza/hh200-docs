# Contributing

## Developing debian package

```sh
$ stack build --copy-bins  # copies hh200 binary to ~/.local/bin/

$ mkdir -p hh200-pkg-ver1/usr/local/bin  # `hh200-pkg-ver1` is an arbitrary package root folder name
$ mkdir -p hh200-pkg-ver1/DEBIAN

$ head hh200-pkg-ver1/DEBIAN/control -n2
Package: <package>
Version: <version>

$ cp ~/.local/bin/hh200 hh200-pkg-ver1/usr/local/bin/
$ chmod 755 hh200-pkg-ver1/usr/local/bin/hh200  # full permissions for the binary

$ dpkg-deb --build hh200-pkg-ver1

??: actually do this
```
Test installing with `sudo dpkg -i hh200-pkg-ver1.deb`. Completely undo installation
with `sudo dpkg --purge loper`.

## Developing docker image
