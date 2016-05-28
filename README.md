# Compile Monero 0.9 on Arch Linux

The example shows how to compile the current github version of [Monero](https://getmonero.org/), as of 18 May 2016, on [Arch Linux](https://www.archlinux.org/). The current version of gcc on Arch is 6.1, which is much newer than that in other distribution, e.g., Ubuntu uses gcc 5.3. Because of [changes gcc 6](https://gcc.gnu.org/gcc-6/changes.html) introduced, normal monero compilation procedure avaliable for Ubuntu will fail. To overcome this, a one line in Monero code needs to be changed to conform to gcc 6.1, and also two treat-warning-as-error errors need to be disabled.

## Dependencies
Before proceeding with the compilation, the following packages are required:

```bash
#install git to download latest Monero source code from github
sudo pacman -Su --needed git

# install dependencies to be able to compile Monero
sudo pacman -Su --needed base-devel cmake boost miniupnpc unbound graphviz doxygen
```

## Compilation

```bash
# download the latest bitmonero source code from github
git clone https://github.com/monero-project/bitmonero.git

# go into bitmonero folder
cd bitmonero/

# apply patch for using Onion Blockchain Explorer (optional)
# https://github.com/moneroexamples/onion-monero-blockchain-explorer
# curl https://raw.githubusercontent.com/moneroexamples/compile-monero-09-on-ubuntu-16-04/master/res/tx_blob_to_tx_info.patch | git apply -v -
#

# apply a patch for gcc 6.1
wget -q -O - https://raw.githubusercontent.com/moneroexamples/compile-monero-09-on-arch-linux/master/fix_value_initialization.patch | git apply  -v -

# compile the release version.
make release CXXFLAGS='-Wno-error=terminate -Wno-error=misleading-indentation'

# alternatively `make` can be used instead of `make release`. This compiles
# the latest, development version of the source code with unit tests.
# It might be unstable as its not officially released.
```

## Installation
After successful compilation, the Monero binaries should be located in `./build/release/bin` as shown below:

```bash
./build/release/bin/
├── bitmonerod
├── blockchain_converter
├── blockchain_dump
├── blockchain_export
├── blockchain_import
├── cn_deserialize
├── connectivity_tool
├── simpleminer
└── simplewallet
```

I usually move the binaries into `/opt/bitmonero/` folder. This can be done in the following way:

```bash
# optional
sudo mkdir -p /opt/bitmonero
sudo mv -v ./build/release/bin/* /opt/bitmonero/
```

This should result in:
```bash
/opt/bitmonero
├── bitmonerod
├── blockchain_converter
├── blockchain_dump
├── blockchain_export
├── blockchain_import
├── cn_deserialize
├── connectivity_tool
├── simpleminer
└── simplewallet
```

Now we can start the Monero daemon, i.e., `bitmonerod`, and let it
download the blockchain and synchronize itself with the Monero network. After that, you can run your the `simplewallet`.

```bash
# launch the Monero daemon and let it synchronize with the Monero network
/opt/bitmonero/bitmonerod

# launch the Monero wallet
/opt/bitmonero/simplewallet
```

## Useful aliases (with rlwrap)
`bitmonerod` and `simplewallet` do not have tab-compliton nor history.
This problem can be overcome using [rlwrap](https://github.com/hanslub42/rlwrap).

```bash
# install rlwrap
sudo pacman -Su rlwrap wget

# download bitmonerod and simplewallet commands files
wget -O ~/.bitmonero/monerocommands_bitmonerod.txt https://raw.githubusercontent.com/moneroexamples/compile-monero-09-on-xubuntu-16-04-beta-1/master/monerocommands_bitmonerod.txt
wget -O ~/.bitmonero/monerocommands_simplewallet.txt https://raw.githubusercontent.com/moneroexamples/compile-monero-09-on-xubuntu-16-04-beta-1/master/monerocommands_simplewallet.txt

# add aliases to .bashrc
echo "alias moneronode='rlwrap -f ~/.bitmonero/monerocommands_simplewallet.txt /opt/bitmonero/bitmonerod'" >> ~/.bashrc
echo "alias monerowallet='rlwrap -f ~/.bitmonero/monerocommands_bitmonerod.txt /opt/bitmonero/simplewallet'" >> ~/.bashrc

# reload .bashrc
source ~/.bashrc
```

With this, we can just start the daemon and wallet simply using
`moneronode` and `monerowallet` commands. `rlwrap` will provide
tab-complition and history for the Monero programs.

## Example screenshot

![Arch Screeshot](https://raw.githubusercontent.com/moneroexamples/compile-monero-09-on-ubuntu-16-04/master/imgs/ubuntu_screen.jpg)


## Monero C++11 development (optional)
If you want to develop your own C++11 programs on top of Monero 0.9,
Monero's static libraries and headers will be needed. Below is shown
how they can be setup for use to write your own C++11 programs based
on Monero. An example of such a program is  [access-blockchain-in-cpp](https://github.com/moneroexamples/access-blockchain-in-cpp).


### Monero static libraries

When the compilation finishes, a number of static Monero libraries
should be generated. We will need them to link against in our C++11 programs.

Since they are spread out over different subfolders of the `./build/` folder, it is easier to just copy them into one folder. I assume that
 `/opt/bitmonero-dev/libs` is the folder where they are going to be copied to.

```bash
# create the folder
sudo mkdir -p /opt/bitmonero-dev/libs

# find the static libraries files (i.e., those with extension of *.a)
# and copy them to /opt/bitmonero-dev/libs
# assuming you are still in bitmonero/ folder which you downloaded from
# github
sudo find ./build/ -name '*.a' -exec cp -v {} /opt/bitmonero-dev/libs  \;
```

 This should results in the following file structure:

 ```bash
/opt/bitmonero-dev/
└── libs
    ├── libblockchain_db.a
    ├── libblocks.a
    ├── libcommon.a
    ├── libcrypto.a
    ├── libcryptonote_core.a
    ├── libcryptonote_protocol.a
    ├── libdaemonizer.a
    ├── libgtest.a
    ├── libgtest_main.a
    ├── liblmdb.a
    ├── libminiupnpc.a
    ├── libmnemonics.a
    ├── libotshell_utils.a
    ├── libp2p.a
    ├── librpc.a
    └── libwallet.a
```

### Monero headers

Now we need to get Monero headers, as this is our interface to the
Monero libraries. Folder `/opt/bitmonero-dev/headers` is assumed
to hold the headers.

```bash
# create the folder
sudo mkdir -p /opt/bitmonero-dev/headers

# find the header files (i.e., those with extension of *.h)
# and copy them to /opt/bitmonero-dev/headers.
# but this time the structure of directories is important
# so rsync is used to find and copy the headers files
sudo rsync -zarv --include="*/" --include="*.h" --exclude="*" --prune-empty-dirs ./ /opt/bitmonero-dev/headers
```

This should results in the following file structure:

```bash
# only src/ folder with up to 3 level nesting is shown

/opt/bitmonero-dev/headers/src/
├── blockchain_db
│   ├── berkeleydb
│   │   └── db_bdb.h
│   ├── blockchain_db.h
│   ├── db_types.h
│   └── lmdb
│       └── db_lmdb.h
├── blockchain_utilities
│   ├── blockchain_utilities.h
│   ├── blocksdat_file.h
│   ├── bootstrap_file.h
│   ├── bootstrap_serialization.h
│   └── fake_core.h
├── blocks
│   └── blocks.h
├── common
│   ├── base58.h
│   ├── boost_serialization_helper.h
│   ├── command_line.h
│   ├── dns_utils.h
│   ├── http_connection.h
│   ├── i18n.h
│   ├── int-util.h
│   ├── pod-class.h
│   ├── rpc_client.h
│   ├── scoped_message_writer.h
│   ├── unordered_containers_boost_serialization.h
│   ├── util.h
│   └── varint.h
# ... the rest not shown to save some space
```

## Other examples
Other examples can be found on  [github](https://github.com/moneroexamples?tab=repositories).
Please know that some of the examples/repositories are not
finished and may not work as intended.

## How can you help?

Constructive criticism, code and website edits are always good. They can be made through github.

Some Monero are also welcome:
```
48daf1rG3hE1Txapcsxh6WXNe9MLNKtu7W7tKTivtSoVLHErYzvdcpea2nSTgGkz66RFP4GKVAsTV14v6G3oddBTHfxP6tU
```
