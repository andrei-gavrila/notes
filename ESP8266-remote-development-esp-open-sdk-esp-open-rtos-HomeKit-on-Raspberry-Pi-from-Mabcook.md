# ESP8266 remote development with esp-open-sdk, esp-open-rtos and HomeKit on Raspberry Pi from Macbook

## Install esp-open-sdk

### References

Overview (command line to build required SDK): https://github.com/maximkulkin/esp-homekit-demo/wiki/Build-instructions-ESP8266

Project documentation: https://github.com/pfalcon/esp-open-sdk

First set of hacks (bash): https://www.studiopieters.nl/esp-homekit-sdk-on-a-raspberry-pi-zero-w/

Mirror for ISL: https://libisl.sourceforge.io (link from https://groups.google.com/g/isl-development/c/JGaMo2VUu_8?pli=1, non removed version https://sourceforge.net/projects/expat/files/expat/2.5.0/)

Fix for cpp errors: https://raw.githubusercontent.com/ChrisMacGregor/esp-open-sdk/master/1001-fix-reload1-compile-error.patch

Fix for gdb compile errors: https://sourceware.org/legacy-ml/gdb-patches/2018-05/msg00863.html

### Install build dependencies

```
andreig@tleilax:~ $ sudo apt-get install make unrar-free autoconf automake libtool gcc g++ gperf flex bison texinfo gawk ncurses-dev libexpat-dev python3-dev python3 python3-serial sed git unzip bash help2man wget bzip2 libtool-bin

andreig@tleilax:~/esp8266 $ git clone --recursive https://github.com/pfalcon/esp-open-sdk.git
```

### Fix missing dependencies (old libraries)

In **crosstool-NG/configure.ac** replace ```|$EGREP '^GNU bash, version (3.[1-9]|4)')``` with ```|$EGREP '^GNU bash, version (3.[1-9]|4|5)')``` (Debian now comes with Bash 5).

In **crosstool-NG/config/companion_libs/expat.in** replace ```default "2.1.0" if EXPAT_V_2_1_0``` with ```default "2.5.0" if EXPAT_V_2_1_0``` (All versions up to 2.5.0 are marked as vulnerable and the tarballs have been removed).

In **crosstool-NG/scripts/build/companion_libs/121-isl.sh** replace ```http://isl.gforge.inria.fr``` with ```https://libisl.sourceforge.io``` (ISL project has moved, this is a mirror location).

Result:

```
andreig@tleilax:~/esp8266/esp-open-sdk/crosstool-NG $ git diff
diff --git a/config/companion_libs/expat.in b/config/companion_libs/expat.in
index 1dff4a79..b26c89bf 100644
--- a/config/companion_libs/expat.in
+++ b/config/companion_libs/expat.in
@@ -16,4 +16,4 @@ config EXPAT_VERSION
     string
 # Don't remove next line
 # CT_INSERT_VERSION_STRING_BELOW
-    default "2.1.0" if EXPAT_V_2_1_0
+    default "2.5.0" if EXPAT_V_2_1_0
diff --git a/configure.ac b/configure.ac
index 5d512fe8..872a73de 100644
--- a/configure.ac
+++ b/configure.ac
@@ -190,7 +190,7 @@ AC_CACHE_VAL([ac_cv_path__BASH],
 AC_CACHE_CHECK([for bash >= 3.1], [ac_cv_path__BASH],
     [AC_PATH_PROGS_FEATURE_CHECK([_BASH], [bash],
         [[_BASH_ver=$($ac_path__BASH --version 2>&1 \
-                     |$EGREP '^GNU bash, version (3\.[1-9]|4)')
+                     |$EGREP '^GNU bash, version (3\.[1-9]|4|5)')
           test -n "$_BASH_ver" && ac_cv_path__BASH=$ac_path__BASH ac_path__BASH_found=:]],
         [AC_MSG_RESULT([no])
          AC_MSG_ERROR([could not find bash >= 3.1])])])
diff --git a/scripts/build/companion_libs/121-isl.sh b/scripts/build/companion_libs/121-isl.sh
index a93d1aad..d5a0f157 100644
--- a/scripts/build/companion_libs/121-isl.sh
+++ b/scripts/build/companion_libs/121-isl.sh
@@ -14,7 +14,7 @@ if [ "${CT_ISL}" = "y" ]; then
 # Download ISL
 do_isl_get() {
     CT_GetFile "isl-${CT_ISL_VERSION}" \
-        http://isl.gforge.inria.fr
+        https://libisl.sourceforge.io
 }

 # Extract ISL
```

### Fix cpp errors

In **crosstool-NG/.build/src/gcc-4.8.5/gcc/reload1.c** replace ```spill_indirect_levels++``` with ```spill_indirect_levels = 1``` (compile error when building with newer gcc).

Result:

```
--- gcc-4.8.5/gcc/reload1.c~	2013-01-21 06:55:05.000000000 -0800
+++ gcc-4.8.5/gcc/reload1.c	2022-01-05 17:18:10.148547719 -0800
@@ -440,7 +440,7 @@
 
   while (memory_address_p (QImode, tem))
     {
-      spill_indirect_levels++;
+      spill_indirect_levels = 1;
       tem = gen_rtx_MEM (Pmode, tem);
     }

```

### Fix gdb compile errors (python upgrade)

Make relevant changes in **crosstool-NG/.build/src/gdb-7.10/gdb/python/python.c** roughly based on:

```
diff --git a/gdb/python/python.c b/gdb/python/python.c
index c29e7d7a6b..89443aee25 100644
--- a/gdb/python/python.c
+++ b/gdb/python/python.c
@@ -1667,6 +1667,14 @@ finalize_python (void *ignore)
   restore_active_ext_lang (previous_active);
 }
 
+#ifdef IS_PY3K
+PyMODINIT_FUNC
+PyInit__gdb (void)
+{
+  return PyModule_Create (&python_GdbModuleDef);
+}
+#endif
+
 static bool
 do_start_initialization ()
 {
@@ -1707,6 +1715,9 @@ do_start_initialization ()
      remain alive for the duration of the program's execution, so
      it is not freed after this call.  */
   Py_SetProgramName (progname_copy);
+
+  /* Define _gdb as a built-in module.  */
+  PyImport_AppendInittab ("_gdb", PyInit__gdb);
 #else
   Py_SetProgramName (progname.release ());
 #endif
@@ -1716,9 +1727,7 @@ do_start_initialization ()
   PyEval_InitThreads ();
 
 #ifdef IS_PY3K
-  gdb_module = PyModule_Create (&python_GdbModuleDef);
-  /* Add _gdb module to the list of known built-in modules.  */
-  _PyImport_FixupBuiltin (gdb_module, "_gdb");
+  gdb_module = PyImport_ImportModule ("_gdb");
 #else
   gdb_module = Py_InitModule ("_gdb", python_GdbMethods);
 #endif
```

And execute a **make distclean** in gdb

```
andreig@tleilax:~/esp8266/esp-open-sdk $ cd crosstool-NG/.build/src/gdb-7.10/
andreig@tleilax:~/esp8266/esp-open-sdk/crosstool-NG/.build/src/gdb-7.10 $ make distclean
```

### Build the SDK

```
andreig@tleilax:~/esp8266/esp-open-sdk $ make toolchain esptool libhal STANDALONE=n
```

## Install esp-open-rtos

### References

Overview (variable to set): https://github.com/maximkulkin/esp-homekit-demo/wiki/Build-instructions-ESP8266

Project documentation: https://github.com/SuperHouse/esp-open-rtos

### Clone the project

```
andreig@tleilax:~/esp8266 $ git clone --recursive https://github.com/Superhouse/esp-open-rtos.git
```

## Install esp-homekit-demo

### References

Overview (variable to set): https://github.com/maximkulkin/esp-homekit-demo/wiki/Build-instructions-ESP8266

Project documentation: https://github.com/maximkulkin/esp-homekit-demo

### Clone the project

```
andreig@tleilax:~/esp8266 $ git clone --recursive https://github.com/maximkulkin/esp-homekit-demo.git
```

## Install esptool

### References

Overview (packages): https://github.com/RavenSystem/esp-homekit-devices/wiki/Install-ESPTool-on-macOS

Project documentation: https://github.com/maximkulkin/esp-homekit-demo


### Install the package

```
andreig@tleilax:~ $ python3 -m pip install esptool --break-system-packages
```

### Patch build system to match the new esptool

In **parameters.mk** replace ```ESPTOOL_ARGS=-fs $(FLASH_SIZE)m -fm $(FLASH_MODE) -ff $(FLASH_SPEED)m``` with ```ESPTOOL_ARGS=-fs $(FLASH_SIZE)MB -fm $(FLASH_MODE) -ff $(FLASH_SPEED)m``` (new version uses M instead of m).

Result:

```
andreig@tleilax:~/esp8266/esp-open-rtos $ git diff
diff --git a/parameters.mk b/parameters.mk
index d133ef5..fc37c74 100644
--- a/parameters.mk
+++ b/parameters.mk
@@ -30,7 +30,7 @@ ESPPORT ?= /dev/ttyUSB0
 ESPBAUD ?= 115200
 
 # firmware tool arguments
-ESPTOOL_ARGS=-fs $(FLASH_SIZE)m -fm $(FLASH_MODE) -ff $(FLASH_SPEED)m
+ESPTOOL_ARGS=-fs $(FLASH_SIZE)MB -fm $(FLASH_MODE) -ff $(FLASH_SPEED)m
 
 
 # set this to 0 if you don't need floating point support in printf/scanf
```

### Test build

Create a **wifi.h** file:

```
andreig@tleilax:~/esp8266/esp-homekit-demo $ cp wifi.h.sample wifi.h
```

Build an example:

```
andreig@tleilax:~/esp8266/esp-homekit-demo $ PATH="/home/andreig/.local/bin:/home/andreig/esp8266/esp-open-sdk/xtensa-lx106-elf/bin:$PATH" SDK_PATH=/home/andreig/esp8266/esp-open-rtos/ FLASH_SIZE=1 make -C examples/led
```

Flash the example:

```
PATH="/home/andreig/.local/bin:/home/andreig/esp8266/esp-open-sdk/xtensa-lx106-elf/bin:$PATH" SDK_PATH=/home/andreig/esp8266/esp-open-rtos/ FLASH_SIZE=1 ESPPORT=rfc2217://localhost:4000?ign_set_control make -C examples/led flash
```

### Monitor the output

Modify the provided rfc2217.py example from pyserial:

```
sudo joe /usr/lib/python3/dist-packages/serial/rfc2217.py
```

Replace the main function with:

```
# simple client test
if __name__ == '__main__':
    import sys
    s = Serial('rfc2217://localhost:4000?ign_set_control', 115200)
    sys.stdout.write('{}\n'.format(s))

    while True:
        try:
            sys.stdout.write(s.read().decode(encoding='ascii', errors='ignore'))
        except KeyboardInterrupt:
            break

    s.close()
```

Start monitoring:

```
python3 /usr/lib/python3/dist-packages/serial/rfc2217.py
```

## Install esptool on macos

### References

Overview: https://docs.espressif.com/projects/esptool/en/latest/esp32s3/esptool/remote-serial-ports.html

Tutorial: https://github.com/RavenSystem/esp-homekit-devices/wiki/Install-ESPTool-on-macOS

Code for server: https://github.com/espressif/esptool/blob/master/esp_rfc2217_server.py

### Install

```
andreig@Andreis-MacBook-Air ~ % python3
xcode-select: note: No developer tools were found, requesting install.
If developer tools are located at a non-default location on disk, use `sudo xcode-select --switch path/to/Xcode.app` to specify the Xcode that you wish to use for command line developer tools, and cancel the installation dialog.
See `man xcode-select` for more details.
```

```
andreig@Andreis-MacBook-Air ~ % python3 -m pip install --upgrade pip
andreig@Andreis-MacBook-Air ~ % python3 -m pip install esptool
```

### Test the install

```
andreig@Andreis-MacBook-Air ~ % python3 -m esptool -p /dev/tty.usbserial-10 chip_id
esptool.py v4.7.0
Serial port /dev/tty.usbserial-10
Connecting....
Detecting chip type... Unsupported detection protocol, switching and trying again...
Connecting....
Detecting chip type... ESP8266
Chip is ESP8266EX
Features: WiFi
Crystal is 26MHz
MAC: 08:3a:8d:d1:c6:45
Uploading stub...
Running stub...
Stub running...
Chip ID: 0x00d1c645
Hard resetting via RTS pin...
```

### Run the serial port forwarder

```
andreig@Andreis-MacBook-Air ~ % python3 ~/Library/Python/3.9/bin/esp_rfc2217_server.py -p 4000 /dev/tty.usbserial-10 
```

### Port forward the serial port forwarder

```
andreig@Andreis-MacBook-Air ~ % ssh -R 4000:localhost:4000 dev-tleilax
```

## Use VSCode

### Simple configuration for homekit

Edit **.vscode/c_cpp_properties.json**:

```
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "../../../esp-open-sdk/lx106-hal/include/**",
                "../../../esp-open-rtos/include/**",
                "../../../esp-open-rtos/libc/xtensa-lx106-elf/include/**",
                "../../../esp-open-rtos/lwip/include/**",
                "../../../esp-open-rtos/lwip/lwip/src/include/**",
                "../../../esp-open-rtos/core/include/**",
                "../../../esp-open-rtos/FreeRTOS/Source/portable/esp8266/**",
                "../../../esp-open-rtos/FreeRTOS/Source/include/**",
                "../../components/common/homekit/include/**",
                "../../**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "c17",
            "cppStandard": "gnu++17",
            "intelliSenseMode": "linux-gcc-arm64"
        }
    ],
    "version": 4
}
```

Build and flash from terminal:

```
andreig@tleilax:~/esp8266/esp-homekit-demo $ PATH="/home/andreig/.local/bin:/home/andreig/esp8266/esp-open-sdk/xtensa-lx106-elf/bin:$PATH" SDK_PATH=/home/andreig/esp8266/esp-open-rtos/ FLASH_SIZE=1 ESPPORT=rfc2217://localhost:4000?ign_set_control make -C examples/led flash
```

Monitor from terminal:

```
andreig@tleilax:~/esp8266/esp-homekit-demo $ python3 /usr/lib/python3/dist-packages/serial/rfc2217.py
```
