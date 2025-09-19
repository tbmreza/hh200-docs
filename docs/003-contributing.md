# Contributing

## Developing
### Developing debian package

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

### Developing docker image

## Roadmap
Criteria for reaching version N.

#### Version 0 Alpha: Hurl alternative
- [ ] honest script interpreter
#### Version 1 Beta: Running distributed
- [ ] containerized (or ssh-based orchestration, undecided)
#### Version 2 Stable: Ecosystem enablement
- [ ] syntax support, LSP in editors (nvim, vscode, emacs)
- [ ] "Copy as hh200" in postman, httpie
- [ ] tui
- [ ] prebuilts in debian, nix, docker hub
- [ ] learnxinyminutes, cheat.sh, awesome lists, LLM RAG
#### Version 3 Maintenance: Benchmarks pushing
- [ ] lexer rewrite
- [ ] dependencies rewrite
