# Build IOG specified version of libblst as a deb package
This document outlines _one_ way to build the IOG specified version of libblst (needed for compiling the cardano-node software) into a Debian package.  Building this library into a deb package makes it easier to install on any Debian or Ubuntu system and have the package management system take care of everything.

This is not an official deb package and it does not do things the official Debian/Ubuntu way.  Furthermore, this deb package does not deliver the proper copyright documents or any manuals.

This deb package will install the software in the following standard locations:

* Libraries get placed in /usr/lib/

## Setup your build environment
I usually create a virtual machine for building software and do everything on it.  This allows me to keep everything separated from my desktop PC.

I also create a separate user ('builder') for building.  This is advisable, even if you choose to build on your desktop PC, so that the installation of any local compilers, or any specific configurations, won't interfere with anything in your normal user account.
```
sudo adduser --gecos '' --disabled-password builder
```

Install build dependencies and some extra requirements
```
sudo apt install build-essential fakeroot devscripts debhelper autoconf-archive d-shlibs pkg-kde-tools git curl automake pkg-config libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev zlib1g-dev make jq libncursesw5 libtool autoconf libncurses-dev libncurses5 libnuma-dev binutils-gold
```

Optionally install llvm and set cc and c++ to use clang
```
sudo apt install llvm clang; \
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100; \
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang++ 100
```
Ubuntu users might need to skip the previous optional step if their Ubuntu version of clang doesn't support optimization flag '-ffat-lto-objects'.  Revert the previous changes with:
```
sudo apt purge llvm clang; \
sudo apt autoremove;
```

Switch to your builder account
```
sudo su - builder
```

The rest of this document assumes you are using your 'builder' account:  
builder@build:~$

IOG specified BLST_VERSION depends on CARDANO_NODE_VERSION. This will set the value to the latest version released for mainnet:
```
CARDANO_NODE_VERSION="$(curl -s https://api.github.com/repos/intersectMBO/cardano-node/releases/latest | jq -r .tag_name)"; \
echo "Latest CARDANO_NODE_VERSION is: ${CARDANO_NODE_VERSION}";
```

The following sequence of commands will remove and recreate the "${HOME}/src/libblst-iog" directory.  Familiarise yourself with the following commands before running them.  You can simply copy and paste the entire list of commands below into a bash terminal to run them in sequence.
```
deb_build_instructions_repo='https://github.com/TerminadaPool/libblst-iog-debian.git'; \

IOHKNIX_VERSION=$(curl https://raw.githubusercontent.com/IntersectMBO/cardano-node/$CARDANO_NODE_VERSION/flake.lock | jq -r '.nodes.iohkNix.locked.rev'); \
echo "iohk-nix version: $IOHKNIX_VERSION"; \
BLST_VERSION=$(curl https://raw.githubusercontent.com/input-output-hk/iohk-nix/$IOHKNIX_VERSION/flake.lock | jq -r '.nodes.blst.original.ref'); \
BLST_VERSION="${BLST_VERSION#v}"; \
echo "Using blst version: ${BLST_VERSION}"; \

package='libblst-iog'; \
basedir="${HOME}/src/${package}"; \

mkdir -p "${basedir}"; \
cd "${basedir}"; \
rm -rf "${package}-${BLST_VERSION}"; \

git clone --depth 1 --branch "v${BLST_VERSION}" https://github.com/supranational/blst "${package}-${BLST_VERSION}"; \
cd "${package}-${BLST_VERSION}"; \

git clone "${deb_build_instructions_repo}" debian; \

debuild -us -uc -b; \

unset deb_build_instructions_repo IOHKNIX_VERSION BLST_VERSION package basedir; \
```

Your deb will be produced in the parent directory: "${HOME}/src/libblst-iog/".  They will be named something like:  
* libblst-iog_0.3.11_amd64.deb

Put these files in your own local debian repository and install them on every machine required using apt.

## Notes
* libblst-iog package is configured to conflict with any standard Debian/Ubuntu libblst package.
* Thus installing libblst-iog will cause the corresponding standard package to be removed.

See: 'control' file for where this is configured.

Expect to see the following lintian errors at the end of the build process:  
> E: libblst-iog: no-copyright-file  


### References
See: https://github.com/input-output-hk/cardano-node-wiki/blob/main/docs/getting-started/install.md

