# Casacore + CASA image analysis

This is a custom version of casacore which includes the CASA image analysis component. It is required as a dependency for the [backend component](https://github.com/CARTAvis/carta-backend/) of the CARTA image viewer.

## Installing measures data

Various parts of casacore depend on measures data. On Ubuntu we recommend installing the `casacore-data` package, either the default Ubuntu version or the package provided by the [KERN suite](https://kernsuite.info/).

Note: Ubuntu `casacore-data` packages now install the data directory in `/var/lib` (although some files may be symbolic links to `/usr/share` or vice versa), and Ubuntu casacore packages expect to find it in that location. The default location used by our source build is still the more universal `/usr/share/casacore/data`, but you can specify a custom location by adding a flag to `cmake`, for example `-DDATA_DIR=/usr/local/share/casacore/data`.

## Ubuntu packages

We provide packages for Ubuntu 20.04 (Focal Fossa) and 18.04 (Bionic Beaver) in [a PPA](https://launchpad.net/~cartavis-team/+archive/ubuntu/carta) on Launchpad. These packages install the headers and shared library files to a custom location in `/opt/carta-casacore`, so that they do not interfere with any existing casacore packages which may be installed from the default Ubuntu repositories or from KERN.

The `casacore-data` package is recommended by our packages and will be installed automatically by default. Use the `--no-install-recommends` flag with `apt` to suppress this behaviour.

```shell
# Add the PPA to your system
sudo add-apt-repository ppa:cartavis-team/carta
sudo apt-get update

# Install CASA dependencies for building the CARTA backend from source
sudo apt-get install carta-casacore-dev

# If you install the binary CARTA backend package, all binary dependencies will be installed automatically.

```

## Install carta-casacore from source

The build dependencies are the same as for [casacore](https://github.com/casacore/casacore#requirements).

Ensure that you specify the correct data directory in a `cmake` parameter if it is not in the default `/usr/share` location, for example `-DDATA_DIR=/var/lib/casacore/data` to specify the latest Ubuntu location.

The CARTA backend source code looks for casacore headers and library files in the location used by the Ubuntu package, so we recommend that you specify the same install location using `CMAKE_INSTALL_PREFIX` if you are building `carta-casacore` from source.

```shell
git clone https://github.com/CARTAvis/carta-casacore.git --recursive

cd ../
mkdir -p build
cd build

cmake .. -DUSE_FFTW3=ON -DUSE_HDF5=ON -DUSE_THREADS=ON -DUSE_OPENMP=ON -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -DBUILD_PYTHON=OFF -DUseCcache=1 -DHAS_CXX11=1 -DCMAKE_INSTALL_PREFIX=/opt/carta-casacore
make -j8
sudo make install
```
