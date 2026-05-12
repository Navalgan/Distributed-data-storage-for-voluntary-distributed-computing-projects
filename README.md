# How To Build BOINC Apps for Android

This document describes how to build BOINC apps for Android.

We assume the app is written in C/C++.
It must be cross-compiled on a non-Android system
(Linux, Windows, or Mac; the instructions here are for Linux).
The BOINC API library (libboincapi) must also be cross-compiled and linked.
The  result of this is a "native mode" executable
for a particular architecture (ARM, MIPS, or Intel) on Android.

Various build systems can be used for this purpose:

* The GNU tools (configure/make).
* Eclipse
* Android Studio

In this document we'll use the GNU tools.

## Cookbook

In the following, we'll assume that on your build system:

* the BOINC source tree is at **~/boinc**.
* the source code for your application is at **~/my_app**.

Download and install the latest Android Native Development Kit (NDK):
http://developer.android.com/tools/sdk/ndk/index.html

Set an environment variable ```NDK_ROOT``` to point to the NDK directory, e.g.
```
export NDK_ROOT="$HOME/android-ndk-r22b"
```

Prepare the Android toolchain by doing
```
cd ~/boinc/android
./buildAndroidBOINC-CI.sh --arch arm64 --component apps
```
This create a directory ```~/android-tc/arm``` containing
compilers, linkers, and libraries for that CPU type.

*You can override the default toolchain location by defining the ```ANDROID_TC``` environment variable.*

## Building FPU versions

To build a version for VFP:
```
-O3 -mhard-float -mfpu=vfp -mfloat-abi=softfp -fomit-frame-pointer
```
To build a version for neon:
```
-O3 -mhard-float -mfpu=neon -mfloat-abi=softfp -fomit-frame-pointer
```

To build project:
```
./configure     --host=aarch64-linux-android     --target=aarch64-linux-android     --with-boinc-platform=aarch64-android-linux-gnu     --disable-server     --disable-manager     --disable-shared     --enable-static     --prefix=$HOME/boinc_android     CC=aarch64-linux-android31-clang     CXX=aarch64-linux-android31-clang++     LD=aarch64-linux-android-ld     AR=llvm-ar     RANLIB=llvm-ranlib     STRIP=llvm-strip     CFLAGS="-DANDROID -D__ANDROID_API__=31 -fPIE -fPIC"     CXXFLAGS="-DANDROID -D__ANDROID_API__=31 -fPIE -fPIC -std=c++11"     LDFLAGS="-pie -static-libstdc++"
```

It's often useful to build a single executable that can support the vfp or neon libraries & instructions,
and then use the BOINC APP_INIT_DATA and HOST_INFO structures
to find the capabilities of the host and call the appropriate optimized routines
(member variable 'p_features' e.g. strstr(aid.host_info.p_features, " neon ").

You can do this by selectively compiling using the above options,
into separate small libraries that you link into the single executable,
and use C++ namespaces to separate similar function calls.
Refer to the boinc/client/Makefile.am and client/whetstone.cpp, and client/cs_benchmark.cpp files
for an example of how to do this.

## Position-independent executables (PIE)
Starting with version 4.1, Android supports **position-independent executables** (PIE).
Starting with version 5.0, PIE is mandatory - Android refuses to run native executables
that are not PIE. **The minimum Android version supported by BOINC is thus 4.1**.

You can build PIE apps by including in your Makefile:
```
LOCAL_CFLAGS += -fPIE
LOCAL_LDFLAGS += -fPIE -pie
```

You should use plan classes so that
* PIE versions are sent only to Android 4.1 or later devices.
* non-PIE versions are sent only to pre-5.0 devices.

For example (using [XML plan class specification](Specifying-plan-classes-in-XML)):
```xml
<plan_class>
  <name>android_arm_pie</name>
  <min_android_version>40100</min_android_version>
</plan_class>

<plan_class>
  <name>android_arm_non_pie</name>
  <max_android_version>49999</max_android_version>
</plan_class>
```

If you have both PIE and non-PIE, change the 49999 to 40099.

## Deploying Android app versions

BOINC on Android uses the following BOINC platform identifier:
"arm-android-linux-gnu"
"x86-android-linux-gnu"
"mipsel-android-linux-gnu"

(how to use plan classes if you have separate FPU versions)
The client reports CPU architecture and capabilities, i.e. VFP and NEON support, to the project server.

## FORTRAN on Android NDK (optional build)

The Android NDK currently does not have a FORTRAN compiler distributed with it.
But it is possible to build GNU Fortran (gfortran) libraries for ARM/Android using the information at
[http://danilogiulianelli.blogspot.co.uk/2013/02/how-to-build-gcc-fortran-cross-compiler.html](http://danilogiulianelli.blogspot.co.uk/2013/02/how-to-build-gcc-fortran-cross-compiler.html)

## Example
Setup the environment:
[samples/example_app/build_android.sh](https://github.com/BOINC/boinc/blob/master/samples/example_app/build_android.sh)

Android Makefile: [samples/example_app/Makefile_android](https://github.com/BOINC/boinc/blob/master/samples/example_app/Makefile_android)

An example of how to adapt a BOINC app's Makefile to compile it for Android
can be found in the AndroidBOINC sources at:
[native/diffs_android/uppercase/Makefile_android](https://github.com/novarow/AndroidBOINC/blob/master/native/diffs_android/uppercase/Makefile_android) (**Note: AndroidBOINC is not supported anymore and out-of-date.**)

