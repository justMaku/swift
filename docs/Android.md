# Getting Started with Swift on Android

The Swift stdlib can be compiled for Android armv7 targets, which makes it
possible to execute Swift code on a mobile device running Android. This guide
explains how to run a simple "Hello, world" program on your Android device.

If you encounter any problems following the instructions below, please file a
bug using https://bugs.swift.org/.

## FAQ

Let's answer a few frequently asked questions right off the bat:

### Does this mean I can write Android applications in Swift?

No. Although the Swift compiler is capable of compiling Swift code that runs
on an Android device, it takes a lot more than just the Swift stdlib to write
an app. You'd need some sort of framework to build a user interface for your
application, which the Swift stdlib does not provide.

Alternatively, one could theoretically call into Java interfaces from Swift,
but unlike as with Objective-C, the Swift compiler does nothing to facilitate
Swift-to-Java bridging.

## Prerequisites

To follow along with this guide, you'll need:

1. A Linux environment capable of building Swift from source. The stdlib is
   currently only buildable for Android from a Linux environment. Before
   attempting to build for Android, please make sure you are able to build
   for Linux by following the instructions in the Swift project README.
2. The latest version of the Android NDK (r11c at the time of this writing),
   available to download here:
   http://developer.android.com/ndk/downloads/index.html.
3. An Android device with remote debugging enabled. We require remote
   debugging in order to deploy built stdlib products to the device. You may
   turn on remote debugging by following the official instructions:
   https://developer.chrome.com/devtools/docs/remote-debugging.

## "Hello, world" on Android

### 1. Downloading (or building) the Swift Android stdlib dependencies

You may have noticed that, in order to build the Swift stdlib for Linux, you
needed to `apt-get install libicu-dev icu-devtools`. Similarly, building
the Swift stdlib for Android requires the libiconv and libicu libraries.
However, you'll need versions of these libraries that work on Android devices.

You may download prebuilt copies of these dependencies, built for Ubuntu 15.10
and Android NDK r11c. Click [here](https://github.com/SwiftAndroid/libiconv-libicu-android/releases/download/android-ndk-r11c/libiconv-libicu-armeabi-v7a-ubuntu-15.10-ndk-r11c.zip)
to download, then unzip the archive file.

Alternatively, you may choose to build libiconv and libicu for Android yourself:

1. Ensure you have `curl`, `autoconf`, `automake`, `libtool`, and
   `git` installed.
2. Clone the [SwiftAndroid/libiconv-libicu-android](https://github.com/SwiftAndroid/libiconv-libicu-android)
   project. From the command-line, run the following command:
   `git clone git@github.com:SwiftAndroid/libiconv-libicu-android.git`.
3. From the command-line, run `which ndk-build`. Confirm that the path to
   the `ndk-build` executable in the Android NDK you downloaded is displayed.
   If not, you may need to add the Android NDK directory to your `PATH`.
4. Enter the `libiconv-libicu-android` directory on the command line, then
   run `build.sh`.
5. Confirm that the build script created `armeabi-v7a/icu/source/i18n` and
   `armeabi-v7a/icu/source/common` directories within your
   `libiconv-libicu-android` directory.

### 2. Building the Swift stdlib for Android

Enter your Swift directory, then run the build script, passing paths to the
Android NDK, as well as the directories that contain the `libicuuc.so` and
`libicui18n.so` you downloaded or built in step one:

```
$ utils/build-script \
    -R \                                # Build in ReleaseAssert mode.
    --android \                         # Build for Android.
    --android-ndk ~/android-ndk-r11c \  # Path to an Android NDK.
    --android-api-level 21 \            # The Android API level to target. Swift only supports 21 or greater.
    --android-icu-uc ~/libicu-android/armeabi-v7a \
    --android-icu-uc-include ~/libicu-android/armeabi-v7a/icu/source/common \
    --android-icu-i18n ~/libicu-android/armeabi-v7a \
    --android-icu-i18n-include ~/libicu-android/armeabi-v7a/icu/source/i18n/
```

### 3. Compiling `hello.swift` to run on an Android device

Create a simple Swift file named `hello.swift`:

```swift
print("Hello, Android")
```

Use the built Swift compiler from the previous step to compile a Swift source
file, targeting Android:

```
$ build/Ninja/ReleaseAssert/swift-linux-x86_64/swiftc \                   # The Swift compiler built in the previous step.
    -target armv7-none-linux-androideabi \                                # Targeting android-armv7.
    -sdk ~/android-ndk-r11c/platforms/android-21/arch-arm \               # Use the same NDK path and API version as you used to build the stdlib in the previous step.
    -L ~/android-ndk-r11c/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a \  # Link the Android NDK's libc++ and libgcc.
    -L ~/android-ndk-r11c/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/lib/gcc/arm-linux-androideabi/4.9 \
    hello.swift
```

This should produce a `hello` executable in the directory you executed the
command. If you attempt to run this executable using your Linux environment,
you'll see the following error:

```
cannot execute binary file: Exec format error
```

This is exactly the error we want: the executable is built to run on an
Android device--it does not run on Linux. Next, let's deploy it to an Android
device in order to execute it.

### 4. Deploying the build products to the device

You can use the `adb push` command to copy build products from your Linux
environment to your Android device. Verify your device is connected and is
listed when you run the `adb devices` command, then run the following
commands to copy the Swift Android stdlib:

```
$ adb push build/Ninja-ReleaseAssert/swift-linux-x86_64/lib/swift/android/libswiftCore.so /data/local/tmp
$ adb push build/Ninja-ReleaseAssert/swift-linux-x86_64/lib/swift/android/libswiftGlibc.so /data/local/tmp
$ adb push build/Ninja-ReleaseAssert/swift-linux-x86_64/lib/swift/android/libswiftSwiftOnoneSupport.so /data/local/tmp
$ adb push build/Ninja-ReleaseAssert/swift-linux-x86_64/lib/swift/android/libswiftRemoteMirror.so /data/local/tmp
$ adb push build/Ninja-ReleaseAssert/swift-linux-x86_64/lib/swift/android/libswiftSwiftExperimental.so /data/local/tmp
```

In addition, you'll also need to copy the Android NDK's libc++:

```
$ adb push ~/android-ndk-r11c/sources/cxx-stl/llvm-libc++/libs/armeabi-v7a/libc++_shared.so /data/local/tmp
```

Finally, you'll need to copy the `hello` executable you built in the
previous step:
```
$ adb push hello /data/local/tmp
```

### 4. Running "Hello, world" on your Android device

You can use the `adb shell` command to execute the `hello` executable on
the Android device:

```
$ adb shell LD_LIBRARY_PATH=/data/local/tmp hello
```

You should see the following output:

```
Hello, Android
```

Congratulations! You've just run your first Swift program on Android.

