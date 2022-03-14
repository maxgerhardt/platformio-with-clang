# PlatformIO with Clang

## Description

This is a test project for integration a LLVM/Clang toolchain with PlatformIO.

As a base project, a TeensyLC board and the Arduino framework is chosen.

This project contains two environments: `teensylc_gcc` and `teensylc_clang` so that the two can be directly compared.

Change between them by using the PIO sidebar -> Project Tasks -> \<environment name\> -> Build / Upload tasks, or the [project envrionment switcher](https://docs.platformio.org/en/latest/integration/ide/vscode.html#project-tasks). The default is GCC so that the baseline can be tested.

## Toolchain download

For successfull compilation, you will need to download the PlatformIO-packaged toolchain package.
* Windows: https://drive.google.com/file/d/1HcVAsVbIxxidfXbI5yAAf4bxpkV4Nr8g/view?usp=sharing
* Linux: https://drive.google.com/file/d/1-AK77wZK5PaBRF3100pZKl9D1qjJldvu/view?usp=sharing
* Mac: ToDo (hard since no binary release provided..)

Download the package for your toolchain, extract it somewhere, and reference the filepath in the `file://<your path here>` part of the `platformio.ini`.

These were created by downloading https://github.com/ARM-software/LLVM-embedded-toolchain-for-Arm/releases/ and repackaging them with a `package.json` which declares the package name, version, etc. (PlatformIO needs this)

Why not the toolchain from https://github.com/llvm/llvm-project/releases/ you might ask? When attempting to use that `clang` with the `-t arm-none-eabi` flags, it failed to find any C standard library headers like `stdlib.h`. On the other hand, the ARM-software provided toolchain has a `lib/clang-runtime` folder for each ARMv6 and ARMv7 variant that includes these headers (and more), along with convenient configuration files (e.g. `armv6m_soft_nofp_nosys.cfg`), so it was used instead.

Feel free to tell my how to use the official LLVM release with a Cortex-M0+ target and the standard C library headers.

## Linking hangs on Windows

This is a peculiar bug in the provided toolchain verison, also reported [here](https://github.com/msys2/MINGW-packages/issues/5231) and [here](https://github.com/msys2/MINGW-packages/issues/61269). The compiled package seems to have some problems with thread notifications and so the linking step appears to hang. **The solution** is to wait a bit (~5 seconds), open the task manager, find the `clang++.exe` process that hangs, kill it, then press the build button in VSCode again, choose "Terminate task", then press the build button yet again. The second linking step should go through most of the time and produce the `firmware.elf`.

## Build errors on Ubuntu 20.04 LTS

```
clang++: error while loading shared libraries: libtinfo.so.5: cannot open shared object file: No such file or directory
```
Recent Ubuntus ship libtinfo6, but clang still depends on libtinfo5 so you may want to install it:
```
> ldd bin/clang++
        [...]
        libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007f9c763a9000)
        libtinfo.so.5 => not found
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f9c761c5000)
        [...]
> sudo apt install libtinfo5
> ldd bin/clang++
        [...]
        libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007fa3a8148000)
        libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007fa3a8118000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fa3a7f36000)
        [...]
```

## Build warnings on Ubuntu 20.04 LTS

Right now a few warnings are popping up with default build settings. You may use build flags like `-Wno-unused-variable` to silence them temporarily, but in general **it's not** recommended (especially for `-Wincompatible-pointer-types` and `-Wcompare-distinct-pointer-types`).

```
/home/ez/.platformio/packages/framework-arduinoteensy/cores/teensy3/mk20dx128.c:693:12: warning: incompatible pointer types initializing 'uint32_t *' (aka 'unsigned int *') with an expression of type 'unsigned long *' [-Wincompatible-pointer-types]
        uint32_t *src = &_etext;
                  ^     ~~~~~~~
/home/ez/.platformio/packages/framework-arduinoteensy/cores/teensy3/mk20dx128.c:694:12: warning: incompatible pointer types initializing 'uint32_t *' (aka 'unsigned int *') with an expression of type 'unsigned long *' [-Wincompatible-pointer-types]
        uint32_t *dest = &_sdata;
                  ^      ~~~~~~~
/home/ez/.platformio/packages/framework-arduinoteensy/cores/teensy3/mk20dx128.c:770:14: warning: comparison of distinct pointer types ('uint32_t *' (aka 'unsigned int *') and 'unsigned long *') [-Wcompare-distinct-pointer-types]
        while (dest < &_edata) *dest++ = *src++;
               ~~~~ ^ ~~~~~~~
/home/ez/.platformio/packages/framework-arduinoteensy/cores/teensy3/mk20dx128.c:1267:20: warning: unused variable 'faultmask' [-Wunused-variable]
        uint32_t primask, faultmask, basepri, ipsr;
                          ^
```


## I get VSCode IntelliSense errors in the Clang configuration

In your `.vscode/c_cpp_properties.json`, you have

```json
            "compilerArgs": [
                "-mthumb",
                "-mcpu=cortex-m0plus",
                "-mno-unaligned-access",
                ""
            ]
```
change it to

```json
            "compilerArgs": [
                "-mthumb",
                "-mcpu=cortex-m0plus",
                "-mno-unaligned-access",
                "--config=armv6m_soft_nofp_nosys",
                ""
            ]
```

This bug comes from PlatformIO (which ultimately comes from not supporting Clang at all) and cannot be influenced in this project.

## Compilation warnings

As of now, there are still bugs to be ironed out in regards to the Teensyduino core and its usage of `uint32_t` which is apparently different to GCC and are likely causing the firmware to not yet work. (Clang's `uint32_t` is `unsigned int` while GCC is `unsigned long int` or something).

```
C:\Users\Max\.platformio\packages\framework-arduinoteensy\cores\teensy3/IntervalTimer.h:68:43: warning: implicit conversion from 'const uint32_t' (aka 'const unsigned int') to 'float' changes value from 178956970 to 178956976 [-Wimplicit-const-int-float-conversion]
                if (microseconds <= 0 || microseconds > MAX_PERIOD) return false;
                                                      ~ ^~~~~~~~~~
C:\Users\Max\.platformio\packages\framework-arduinoteensy\cores\teensy3/DMAChannel.h:800:72: warning: all paths through this function will call itself [-Winfinite-recursion]
        void destinationCircular(volatile unsigned int p[], unsigned int len) {
                                                                              ^
```

I am not yet sure how to solve these properly and retain compatibility to other libraries that may use `uint32_t`.

**EDIT**: A fix was attempted with up-promoting the types and changing some function prototypes around. Not tested on real hardware.
