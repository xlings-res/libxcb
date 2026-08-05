# libxcb

xlings-res payload for **`libxcb`** — part of the xlings graphics stack (T2).

Built from source **inside an xlings subos**, against that subos's glibc, so
the result depends on nothing from whatever host it is installed on. That is
the whole point: a payload linked against a build machine's glibc works there
and fails elsewhere, which is
[mcpp#352](https://github.com/mcpp-community/mcpp/issues/352).

| | |
|---|---|
| version | `1.17.0` |
| upstream | https://xorg.freedesktop.org/archive/individual/lib/libxcb-1.17.0.tar.xz |
| sha256 | `561b2f8f6d1452f211cdb9023efc3ecde8c365f1eba181bd2d183357b85f484a` |
| recipe | [`xim-pkgindex/pkgs/l/libxcb.lua`](https://github.com/openxlings/xim-pkgindex/blob/main/pkgs/l/libxcb.lua) |

## Build

```bash
git clone https://github.com/openxlings/xim-pkgindex && cd xim-pkgindex
xlings subos new gfxbuild
xlings subos use gfxbuild --global
xlings install gcc make ninja python cmake zlib expat -y

export XLINGS_GFX_SUBOS=gfxbuild
bash .agents/tools/graphics/tiers.sh T2     # or build-libllvm.sh for libllvm
```

Standard autotools/meson configuration; the harness supplies `--prefix=/usr --libdir=lib` and the subos toolchain.

## What the harness guarantees

`build-in-subos.sh` refuses a payload that reaches back to the host, checking
for both kinds of leak before packaging:

* an RPATH naming anything outside the payload;
* an absolute host prefix baked into a `.pc`, `.la` or `*-config` file — which
  breaks nothing now and breaks the **next** package, by pointing its
  `configure` at `/usr`.

`PKG_CONFIG_LIBDIR` points only at the subos, so a dependency that is not
packaged yet fails `configure` loudly instead of being satisfied quietly by a
host copy.

### Build-time dependency on xcb-proto's Python module

`libxcb` generates its protocol bindings from the XML in `xcb-proto`, using
that package's `xcbgen` Python module. Both the module and the `.pc` that
locates the XML must be visible to the build.

The `.pc` records `prefix=/usr` — correct for a relocatable payload, wrong
during the build, where the files live under the subos. `build-in-subos.sh`
rewrites the prefix in the **staged** copy only; the shipped tarball keeps
`/usr`. Without that rewrite the build fails looking for `//usr/share/xcb/`,
a path on the host root.


## Verifying the stack

```bash
bash .agents/tools/graphics/selfcontained-check.sh
```

Runs a GL client under bwrap with **no `/usr` and no `/lib`** — only the subos,
`/dev/dri` and a read-only `/sys` — and asserts it renders correct pixels and
that the renderer is *not* the host's driver. That last assertion is the only
thing separating a self-contained stack from one quietly using the host's:
both produce identical output.
