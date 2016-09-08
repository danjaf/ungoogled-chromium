# ungoogled-chromium

**A Google Chromium variant focusing on removing Google integration, enhancing privacy, and adding features**

* [Features](#features)
    * [Supported platforms and distributions](#supported-platforms-and-distributions)
* [Download pre-built packages](#download-pre-built-packages)
* [Getting the source code](#getting-the-source-code)
* [Design and implementation](#design-and-implementation)
* [Building](#building)
* [Contributing](#contributing)
    * [Pull requests](#pull-requests)
* [Credits](#credits)
* [License](#license)

## Features

In addition to features provided by [Iridium Browser](//iridiumbrowser.de/) and [Inox patchset](//github.com/gcarq/inox-patchset), the following is also included:
* Replace many web domains in the source code with non-existent alternatives ending in `qjz9zk` (known as domain substitution)
* Strip binaries from the source code (known as source cleaning)
    * This includes all pre-built executables, shared libraries, and other forms of machine code. They are substituted with system or user-provided equivalents, or built from source.
    * However some data files (e.g. `icudtl.dat` for Unicode and Globalization support and `*_page_model.bin` that define page models for the DOM Distiller) are left in as they do not contain machine code and are needed for building.
* Disable functionality specific to Google domains
* Disable searching in the Omnibox
    * Fixes minor annoyances when the input is a resolvable address, not a search query.
    * Will be configurable from the UI in the future since this breaks many people's workflows
* Disable automatic formatting of URLs in Omnibox (e.g. stripping `http://`, hiding certain parameters)
* Disable JavaScript dialog boxes from showing when a page closes (onbeforeunload events)
    * Bypasses the annoying dialog boxes that spawn when a page is being closed
* Added menu item under "More tools" to clear the HTTP authentication cache on-demand
* Force all pop-ups into tabs
* Disable intranet redirect detector
    * Prevents unnecessary invalid DNS requests to the DNS server.
    * This breaks captive portal detection, but captive portals still work.
* Add more URL schemes allowed for saving
    * Note that this generally works only for the MHTML option, since an MHTML page is generated from the rendered page and not the original cached page like the HTML option.
* (Iridium Browser feature change) Prevent URLs with the `trk:` scheme from connecting to the Internet
    * Also prevents any URLs with the top-level domain `qjz9zk` (as used in domain substitution) from attempting a connection.
* (Iridium and Inox feature change) Prevent pinging of IPv6 address when detecting the availability of IPv6
* Support for building Debian and Ubuntu packages
    * Creates a separate package `chrome-sandbox` for the SUID sandbox
        * Not necessary to install if the kernel option `unprivileged_userns_clone` is enabled
* Windows support with additional changes:
    * Build `wow_helper.exe` from source instead of using the pre-built version
    * Build `swapimport.exe` from source instead of downloading it from Google (requires [customized syzygy source code](//github.com/Eloston/syzygy))
    * Build `yasm.exe` from source instead of using the pre-built version
    * Use user-provided building utilities instead of the ones bundled with Chromium (currently `gperf` and `bison`)
    * Do not set the Zone Identifier on downloaded files (which is a hassle to unset)

**DISCLAIMER: Although it is the top priority to eliminate bugs and privacy-invading code, there will be those that slip by due to the fast-paced growth and evolution of the Chromium project.**

### Supported platforms and distributions
* Debian
* Ubuntu
* Windows
* Mac OS

## Download pre-built packages

[Downloads for the latest release](//github.com/Eloston/ungoogled-chromium/releases/latest)

[List of all releases](//github.com/Eloston/ungoogled-chromium/releases)

The release versioning scheme follows that of the tags. See the next section for more details.

## Getting the source code

Users are encouraged to use [one of the tags](//github.com/Eloston/ungoogled-chromium/tags). The `master` branch is not guaranteed to be in a working state.

Tags are versioned in the following format: `{chromium_version}-{release_revision}` where

* `chromium_version` is the version of Chromium used in `x.x.x.x` format, and
* `release_revision` is a number indicating the version of ungoogled-chromium for the corresponding Chromium version.

## Design and implementation

Features are implemented through a combination of build flags, patches, and a few configuration files for scripts. All of these settings are stored in the `resources` directory. The `resources` directory contains the `common` directory, which has such files that apply to all platforms. All other directories, named by platform, contain additional platform-specific data. Most of the features, however, are stored in the `common` directory.

There are currently two automated scripts that process the source code:
* Source cleaner - Used to clean out binary files (i.e. do not seem to be human-readable text files, except a few required for building)
* Domain substitution - Used to replace Google and other domains in the source code to eliminate communication not caught by the patches and build flags.

These are the general steps that ungoogled-chromium takes to build:

1. Get the source code archive in `.tar.xz` format via `https://commondatastorage.googleapis.com/` and extract it into `build/sandbox/`
    * Also download any additional non-Linux dependencies for building on non-Linux platforms, since the `.tar.xz` is generated on a Linux system
2. Run source cleaner (done during source archive extraction)
    * Optional, enabled by default
2. Run domain substitution
    * Optional, enabled by default
2. Copy patches into `build/patches/` and apply them
    * If domain substitution was run earlier, then the patches will pass through domain substitution first
3. Configure the build utilities and run meta-build configuration (i.e. GYP, not GN. See [Issue #16](//github.com/Eloston/ungoogled-chromium/issues/16))
4. Build (via 'ninja')
5. Generate binary packages and place them in `build/`

Here's a breakdown of what is in a resources directory:
* `cleaning_list` - (Used for source cleaning) A list of files to be excluded during the extraction of the Chromium source
* `domain_regex_list` - (Used for domain substitution) A list of regular expressions that define how domains will be replaced in the source code
* `domain_substitution_list` - (Used for domain substitution) A list of files that are processed by `domain_regex_list`
* `extra_deps.ini` - Contains info to download extra dependencies needed for the platform but not included in the main Chromium source archive
* `gn_args.ini` - A list of GN arguments to use for building. (Currently unused, see [Issue #16](//github.com/Eloston/ungoogled-chromium/issues/16))
* `gyp_flags` - A list of GYP flags to use for building.
* `patches/` - Contains patches. `common/patches` directory contains patches that provide the main features of ungoogled-chromium (as listed above) and can be applied on any platform (but are not necessarily designed to affect all platforms). However, other `patches/` directories in other platform directories are platform-specific. The contents of `common/patches` are explained more in-depth below.
    * `patch_order` - The order to apply the patches in. Patches from `common` should be applied before the one for a platform.

All of these files are human-readable, but they are usually processed by the Python building system. See the Building section below for more information.

Here's a breakdown of the `common/patches` directory:
* `ungoogled-chromium/` - Contains new patches for ungoogled-chromium. They implement the features described above.
* `iridium-browser` - Contains a subset of patches from Iridium Browser.
    * Patches are not touched unless they do not apply cleanly onto the version of Chromium being built
    * Patches are from the `patchview` branch of Iridium's Git repository. [Git webview of the patchview branch](//git.iridiumbrowser.de/cgit.cgi/iridium-browser/?h=patchview)
* `inox-patchset/` - Contains a modified subset of patches from Inox patchset.
    * Patches are from [inox-patchset's GitHub](//github.com/gcarq/inox-patchset)
    * [Inox patchset's license](//github.com/gcarq/inox-patchset/blob/master/LICENSE)
* `debian/` - Contains patches from Debian's Chromium.
    * These patches are not Debian-specific. For those, see the `resources/debian/patches` directory

## Building

[See BUILDING.md](BUILDING.md)

## Contributing

Contributers are welcome!

Use the [Issue Tracker](//github.com/Eloston/ungoogled-chromium/issues) for problems, suggestions, and questions.

### Pull requests

Pull requests are also welcome. Here are some guidelines:
* Changes that fix certain configurations or add small features and do not break compatibility are generally okay
* Larger changes, such as those that change `buildlib`, should be proposed through an issue first before submitting a pull request.
* When in doubt, propose the idea through an issue first.

## Credits

[Iridium Browser](//iridiumbrowser.de/)

[Inox patchset](//github.com/gcarq/inox-patchset)

[Debian for build scripts](//tracker.debian.org/pkg/chromium-browser)

[The Chromium Project](//www.chromium.org/)

## License

GPLv3. See [LICENSE](LICENSE)
