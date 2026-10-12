---
title: "Customizing the sstate Cache and Download Directories in Yocto"
description: "What DL_DIR and SSTATE_DIR do, and how sharing them between projects saves hours of build time."
date: 2026-10-12 08:55:00 +0100
categories: [Yocto]
tags: [yocto, bitbake, sstate-cache, dl-dir, local-conf, build-time]
---

## Understanding these directories

### Download directory (`DL_DIR`)

This directory stores all the source tarballs and Git clones fetched by BitBake during the build. At the beginning of a build (the `do_fetch` task), BitBake parses the `SRC_URI` variable and downloads the source code from each URL into the download directory.

- **Default location:** `${BUILDDIR}/downloads`
- **If you delete it:** BitBake re-downloads everything on the next build.
- **Variable:** `DL_DIR`

### Sstate cache directory (`SSTATE_DIR`)

The sstate cache (*shared state cache*) holds the results of each build task. Each time BitBake builds a package, it stores the compiled output (binaries, libraries, headers, etc.) in this directory. If another project needs the same recipe with identical metadata, BitBake restores it from the cache instead of rebuilding it. This is what makes BitBake rebuilds fast.

- **Default location:** `${BUILDDIR}/sstate-cache`
- **If you delete it:** BitBake rebuilds everything on the next build.
- **Variable:** `SSTATE_DIR`

| | `DL_DIR` | `SSTATE_DIR` |
|---|---|---|
| Stores | Source tarballs and Git clones | Compiled task results |
| Filled by | `do_fetch` | Every task that produces output |
| Default | `${BUILDDIR}/downloads` | `${BUILDDIR}/sstate-cache` |
| After deletion | Everything is downloaded again | Everything is rebuilt |

## Sharing the download sources and the sstate cache

Sharing `DL_DIR` and `SSTATE_DIR` is a small configuration tweak that can save hours of build time and make your Yocto workflow much more efficient.

This is useful when several build directories target similar distributions (for example, different branches of the same project): the shared sstate cache lets BitBake reuse most of the previously built artifacts.

Even when projects target distinct architectures such as ARM and RISC-V, sharing remains useful, because native (host) packages are also cached and reused.

To share these directories between several projects, set `DL_DIR` and `SSTATE_DIR` in `conf/local.conf` of each build directory to your own paths.

Example in `local.conf`:

```bitbake
DL_DIR ?= "/home/user/yocto-data/downloads"
SSTATE_DIR ?= "/home/user/yocto-data/sstate-cache"
```

## Useful commands

Check the values of `DL_DIR` and `SSTATE_DIR`:

```bash
bitbake-getvar --value DL_DIR
bitbake-getvar --value SSTATE_DIR
```

Clean the cached objects of a recipe:

```bash
bitbake -c cleansstate <recipe>
bitbake -c cleanall <recipe>
```

> `cleansstate` removes the recipe's build output and its sstate objects. `cleanall` goes further and also deletes the recipe's downloaded sources from `DL_DIR`, so use it with care.
{: .prompt-warning }
