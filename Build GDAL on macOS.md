Build Instructions: https://gdal.org/en/stable/development/building_from_source.html

```sh
cd gdal
mkdir build
cd build
cmake -DCMAKE_DISABLE_FIND_PACKAGE_Arrow=ON -DGDAL_USE_LIBKML=OFF -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
cmake --build .
cmake --build . --target install
```

