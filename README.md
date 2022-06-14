# libcxx-provider README

## What is this?

libcxx-provider is almost a stub project that only provides libc++_shared.so to apps.

If you build somewhat complex Android native application using Android NDK and are suffered from:

> java.lang.UnsatisfiedLinkError: dlopen failed: library "libc++_shared.so" not found: needed by ...

...then this libcxx-provider is your savior.

libcxx-provider is a [Prefab](https://google.github.io/prefab/) package, and published via [Maven Central](https://repo1.maven.org/maven2/dev/atsushieno/).


## Usage

In your app project, add the following lines to your `build.gradle.kts` (or `build.gradle`):

```
dependencies {
    runtimeOnly("dev.atsushieno:libcxx-provider:21.4.7075529")
}
```

`runtimeOnly` would look unfamiliar to most of you - it indicates that the package contents are thrown into the outputs without building anything. You can use `implementation` instead of `runtimeOnly`, and optionally use the String value of the NDK version the `libc++_shared.so` comes from, but it involves extra compilcation steps from C++ and Kotlin sources.

The package version number actually indicates the NDK version that provides `libc++_shared.so_`. Version `21.4.7075529` is built with NDK r21e (`21.4.7075529`).

At the moment, we will have the following versions (actually this document is written before publishing those packages, so there may be some gap):

- `21.4.7075529` (r21e)
- `22.1.7171670` (r22b)
- `23.2.8568313` (r23c)
- `24.0.8215888` (r24)
- `25.0.8528842` (r25-RC4)

(Using NDK versions directly is kind of reckless but it would be better than `0.21.4` etc. because we cannot ship beta/RC versions. Hopefully I make no mistake on them. In the worst case we will come up with further dotted revisions.)


## Rationale

### ODR

Android NDK comes with LLVM C++ standard library i.e. libc++ (which is comparable to GCC libstdc++). Whenever we write and compile C++ code, we most definitely need the STL.

One important rule you have to follow is [ODR - One Definition Rule](https://developer.android.com/ndk/guides/cpp-support#one_stl_per_app). We cannot use multiple C++ runtimes.

Unlike C standard library i.e. libc (bionic), libc++ is not shipped as part of Android platform system libraries, so we cannot reference libc.so to load at run-time from the system (where ODR would not matter) but package libc++ in our apps instead.

libc++ can be linked either statically or dynamically, and for simple apps that only link every native code bits into one application library (.so) statically, things are simple. But if you have two or more shared libraries that linked libc++ statically, [that goes problematic](https://developer.android.com/ndk/guides/cpp-support#static_runtimes). You can easily break the ODR rule.

Therefore we are mostly supposed to use the shared version i.e. `libc++_shared.so` in our libraries.

ODR should not be specific to C++. For example, there should be only one `kotlin-stdlib-*.jar` throughout the apk, but Gradle Kotlin plugin would resolve those conflicts to only the latest version. That does not happen to C++/NDK build systems (CMake or ndk-build).

### Excluding libc++_shared.so from every AAR

The ODR rule is simple in principle, but we are suffered from various pitfalls.

A popular problem is "More than one file was found with OS independent path 'lib/x86/libc++_shared.so'" etc. It happens when more than one AARs package the same library (even if they are identical).

Whenever we build an AAR that contains a native library that depends on another (base) library (by adding its container AAR dependency), it is also packaged into the AAR. `libc++_shared.so` can be therefore easily inflated everywhere. We can exclude those dependency `*.so` files from AARs by e.g.:

```
    packagingOptions {
        jniLibs.excludes.add("**/libc++_shared.so")
    }
```

This way, we don't have to worry about `libc++_shared.so` duplicates.

When your application project that references such AARs is being built using NDK, it will resolve `libc++_shared.so` and package into the apk/aab.

### Problem on using native-bound libraries without C++ compilation

I (@atsushieno) build [an audio plugin framework](https://github.com/atsushieno/android-audio-plugin-framework/) that comes with a couple of framework library AARs and they are used by various plugin apps. Some of those AARs come with the native libraries, and they build up a dependency tree.

Those library AARs are in general designed for apps to be usable without compiling native code. JVM interop projects maybe also good use cases.

This however uncovers an interesting problem. Those apps often don't have to involve `externalNativeBuild`, then no native compilation happens. In such case, only those native libraries from AARs are packaged into the apk/aab. Now you may have noticed - `libc++_shared.so` is not in any of the AARs, and there is no further native compilation involved.

What happens then? "java.lang.UnsatisfiedLinkError: dlopen failed: library "libc++_shared.so" not found" ...

It is [what I experienced](https://github.com/atsushieno/android-audio-plugin-framework/issues/109) and what I wanted to resolve with this package. By adding this simplest AAR package, the missing `libc++_shared.so` piece is filled in my apps.

IMHO The Android NDK team should distribute a package like this, just like how Kotlin team does for `kotlin-stdlib` (or the .NET team for CoreCLR/CoreFX artifacts), but so far anyone could do this, hence I am on it now.


## Hacking

### How to build for newer NDK version

Find and replace those lines that specifies the version:

```
/sources/libcxx-provider$ git grep -n 21.4 | grep -v README.md
libcxx-provider/build.gradle:18:        versionName "21.4.7075529" // It has to align with ndkVersion
libcxx-provider/build.gradle:26:    ndkVersion "21.4.7075529"
libcxx-provider/src/main/cpp/package-info.cpp:7:            const char *ndk_version = "21.4.7075529";
libcxx-provider/src/main/java/dev/atsushieno/libcxx/LibCXX.java:4:    String ndkVersion = "21.4.7075529";
```

Then run `./gradlew build publishToMavenLocal` to build and install to your local Maven repository, or `./gradle build publish` to publish to the registered Maven server (note that only the package maintainers can perform it).


## License

libcxx-provider sources are distributed under the MIT license.

The corresponding Maven packages are distributed under the Apache V2 licanse.
