
## build casacore + imageanalysis

This works for me...

```
-bash-4.2$ mkdir build
-bash-4.2$ cd build
-bash-4.2$ cmake -DUseCcache=1 -DHAS_CXX11=1 -DCMAKE_CXX_COMPILER=/opt/rh/devtoolset-4/root/usr/bin/g++ -DCMAKE_C_COMPILER=/opt/rh/devtoolset-4/root/usr/bin/gcc -DCMAKE_Fortran_COMPILER=/opt/rh/devtoolset-4/root/usr/bin/gfortran -DBoost_NO_BOOST_CMAKE=1 -DBUILD_TESTING=OFF -DCMAKE_INSTALL_PREFIX=../root -DCMAKE_BUILD_TYPE=Debug ..
-bash-4.2$ make
```
