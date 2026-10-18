# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is one of the 32 formula forks under `../` (`github.com/jbox-web/sysctl-formula`).
The parent `../CLAUDE.md` describes the shared toolchain, CI and conventions; this file
only records what differs here. **This fork does not follow the layout the parent
describes** — no `osfamilymap.yaml` chain, no `package/` / `config/` / `service/`
split, no `_mapdata` reference dumps.

## Commands

```bash
bundle install
bin/kitchen list                                  # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2         # destroy + converge + verify + destroy
pre-commit run --all-files
```

There is a single suite, `default`. Its pillar is `test/salt/pillar/sysctl.sls`
(one flat param, `vm.swappiness: 20`).

Pre-commit here passes the matching file list to rubocop and yamllint (no
`always_run`), so a hook run on staged files only lints those files.

## What the formula does

Two states, both flat files under `sysctl/`, included by `sysctl/init.sls`:

- `sysctl.package` — `pkg.installed` of `mapdata.sysctl.pkg`, with `failhard: True`
  (a package failure aborts the whole highstate).
- `sysctl.param` — one `sysctl.present` per parameter. Three pillar shapes are
  accepted, tried in this order:
  1. `sysctl:params:<key>: <value>` (flat) or `sysctl:params:<key>: {value, config}`;
  2. legacy `sysctl:lookup:params2:<key>: {value, config}`;
  3. legacy `sysctl:lookup:params: [{name, value, config}]`.

  `config` is a file name joined to `config.location` (`/etc/sysctl.d`). Without it,
  Salt picks its own default file (`/etc/sysctl.d/99-salt.conf` under systemd).
  Nothing removes a parameter dropped from the pillar.

## `map.jinja` — the mapstack variant

`map.jinja` is upstream's newer template map, built on `libmapstack.jinja`
(+ `libmatchers.jinja`, `libsaltcli.jinja`). It runs in two passes:

1. It loads `sysctl/parameters/map_jinja.yaml`, which **replaces** the default source
   list with: `Y:G@os_family`, `C:SUB@sysctl:lookup`, `C:SUB@sysctl`.
2. It merges `parameters/defaults.yaml`, then `parameters/os_family/<os_family>.yaml`,
   then the pillar/config `sysctl:lookup` and `sysctl` keys.

The `SUB` option merges the pillar under `mapdata.sysctl` (with `:lookup` stripped),
which is why states read `mapdata.sysctl.*`. Every parameter file wraps its content
in a top-level `values:` key. Per-distro defaults go in
`parameters/os_family/<family>.yaml` (Debian, Gentoo and Suse exist; the default
package is `procps-ng`, Debian's is `procps`). A `sysctl/post-map.jinja` hook is
included if present (none exists).

## Tests

- `test/integration/default/controls/package_spec.rb` is the only control: it checks
  the package. Nothing verifies the `sysctl.param` result.
- `sysctl._mapdata` runs in `state_top`, but there are no
  `test/integration/default/files/_mapdata/*.yaml` reference dumps and no control
  compares them.

## Stale upstream text

`docs/README.rst` says `sysctl.param` manages `sysctl.conf` from a template in
`sysctl/files`; there is no template and no `files/` directory. Its testing example
(`debian-9-2019-2-py3`) predates the current platforms in `kitchen.yml`.
