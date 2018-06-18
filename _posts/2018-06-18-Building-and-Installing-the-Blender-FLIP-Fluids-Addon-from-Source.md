---
title:  "Building and Installing the Blender FLIP-Fluids Addon from Source"
date:   2018-06-18
comments: true
---
Earlier this year there was an exciting new add-on released for Blender, allowing the use of Fluid-implicit particle (FLIP) fluid simulation. FLIP is an efficient method for complex fluid simulation, including foam, bubbles, and spray. Compared to Blender's built-in fluid simulator, this method is faster and generates truly remarkable results. The only catch is that the plugin itself costs $76 on the [Blender market](https://www.blendermarket.com/products/flipfluids). While this isn't particularly expensive, as a poor student who mainly uses Blender to procrastinate, it is still a bit too much money. Fortunately, the source code for the plugin is [freely available on GitHub](https://github.com/rlguy/Blender-FLIP-Fluids), which means I could procrastinate even more by trying to build it myself. And now that I've successfully done that I figured I could procrastinate once again by documenting my process.

## Requirements
- OpenCL libraries from your graphics card vendor
    - For Nvidia, this is found in the [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads)
- CMake for configuration and building
- A compatible toolchain for building
    - I'm on windows, so I use Microsofts [Build Tools for Visual Studio 2017](https://www.visualstudio.com/downloads/#build-tools-for-visual-studio-2017)
    - MinGW should work too, but I haven't tried

## Getting and Compiling the Source
The exact command for building depends on what toolchain and platform you use. In my case (Visual Studio on 64-bit Windows) I used the following:

```powershell
git clone https://github.com/rlguy/Blender-FLIP-Fluids.git
cd ./Blender-FLIP-Fluids
cmake . -G "Visual Studio 15 2017 Win64"
cmake --build .
```

If it compiles out of the box without throwing errors you can skip directly to the installation step. I had to tweak a few things to get it to build properly.

First of all the compiler complains that `min` and `max` are not members of `std`. This means `algorithm` isn't included in a few places where it is needed. To fix this add `#include <algorithm>` to the includes in the offending files. In my case this was `/src/engine/meshlevelset.h`, `/src/engine/opencl_bindings/clcpp.h` and `/src/engine/c_bindings/openclutils.h`.

The second issue had to do with OpenCL. For some reason, the OpenCL library location is explicitly set to the DLL `C:/Windows/System32/OpenCL.dll` rather than the default OpenCL .lib file (In my case `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v9.2\lib\x64\OpenCL.lib`). This causes the linker to throw a bunch of `LNK1107` errors. It can be easily fixed by removing the section in `CMakeLists.txt`.

**Change this:**

```cmake
if(WITH_OPENCL)
    if(MSVC OR MINGW)
        Set to OpenCL ICD loader instead of vendor library
        set(OpenCL_LIBRARY "C:/Windows/System32/OpenCL.dll")
    endif()

    # Allows use os clCreateCommandQueue in OpenCL versions >= 2.0
    add_definitions(-DCL_USE_DEPRECATED_OPENCL_1_2_APIS)

    add_definitions(-DWITH_OPENCL=1)
    include_directories(src ${OpenCL_INCLUDE_DIRS})
else()
    add_definitions(-DWITH_OPENCL=0)
endif()
```

**To this:**
```cmake
if(WITH_OPENCL)
    # Allows use os clCreateCommandQueue in OpenCL versions >= 2.0
    add_definitions(-DCL_USE_DEPRECATED_OPENCL_1_2_APIS)

    add_definitions(-DWITH_OPENCL=1)
    include_directories(src ${OpenCL_INCLUDE_DIRS})
else()
    add_definitions(-DWITH_OPENCL=0)
endif()
```

For me, this was enough to get it to build properly. All I had to do was re-run the build command.

```powershell
cmake --build .
```

If all goes well you should now have a `bl_flip_fluids` directory.

# Installation
To install the addon, simply copy `bl_flip_fluids/flip_fluids_addon` to the Blender addons directory (In my case `C:\Program Files\Blender Foundation\Blender\2.79\scripts\addons`).

To start playing with FLIP-Fluids in Blender, simply enable the addon in user preferences. Make sure it can detect your GPU compute device.

![UserPrefs]({{ site.url }}/assets/images/posts/2018-06-18-Building-and-Installing-the-Blender-FLIP-Fluids-Addon-from-Source/UserPrefs.png)