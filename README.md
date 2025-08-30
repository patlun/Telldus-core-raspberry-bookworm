# installing telldus-core in Raspberry OS Bookworm

## Background

My notes about how to installing [telldus-core](https://github.com/rubund/telldus-core) in Raspberry OS Bookworm as my old raspbian host died and I needed to reinstall, including telldus-core. Don't forget your backups people!

I used an version of telldus-core I had on disk (telldus-core 2.1.3 beta) as that was the version I used last time and I could not find any newer. 

I needed to do some small changes in the code to make it work 2025.

## For the reader.
- This is my notes from setting up the Tellstick duo for my use case, that is to read sensor data.
- I have tried to add relevant commands below, but I can have missed something, I expect the reader to have an intermediate understanding Linux.
- I will not tell you how to install and configure your Raspberry Pi.
- This worked for me
- There will be warnings while compiling telldus-core. I have not looked into them
- I hope you will be able to grok this if needed.

## Prerequisites
### Hardware and operation system
- Raspberry Pi 3B+, makes it possible to boot from USB. You can get Raspberry Pi 3B (without plus) to [boot from USB too](https://www.instructables.com/Booting-Raspberry-Pi-3-B-With-a-USB-Drive/)
- Sandisk Ultra Fit USB stick. I use this because of the small physical format, other brands have similar ones 
-  Tellstick duo
- Raspberry OS Bookworm Lite installed and up to date. I used the image from 2025-05-13

### Setup
- ssh and wifi enabled
- Local user created and ssh key copied for password-less logon.

## Installation of telldus-core

### Create temporary directory
Create a directory in your home directory in the Raspberry Pi and jump there.
```
mkdir Telldus-core
cd Telldus-core
```

### Install dependencies
I installed a bunch of programs and librariers recommended in the sources.
```
sudo apt-get install libftdi1 libftdi-dev git cmake autoconf automake  autoconf-archive gnu-standards autoconf-doc libtool gettext m4-doc flex bison flex-doc autopoint gettext-doc libasprintf-dev libgettextpo-dev
```
Some are used for development (build of telldus-core and libconfuse) some are used in runtime.

### Install libconfuse
As libconfuse is used by telldus-core, but not in the main repos for Raspberry OS I decided to compile it myself. 

I cloned the repo from github and used the standard compilation process.
```
git clone https://github.com/libconfuse/libconfuse.git
cd libconfuse
```
As I cloned the repo I needed to run autogen.sh, if you download a tar-ball you can start with the configure.

```
./autogen.sh
./configure
make -j9
sudo make install
```

### install telldus-core

#### Download
I was using a file I already had downloaded. But you can download that from.

```
wget http://download.telldus.com/pool/main/t/telldus-core/telldus-core_2.1.3-beta1.orig.tar.gz
```

For a more stable version:
```
wget http://download.telldus.com/TellStick/Software/telldus-core/telldus-core-2.1.2.tar.gz
```
I have not tested to install 2.1.2.

#### Unpack
```
tar xzf telldus-core_2.1.3-beta1.orig.tar.gz 
```
Jump into the code 
```
cd telldus-core-2.1.3-beta1/
```

#### Needed changes.

##### CMakeList.txt
I removed the doxygen part, starting at line 59.
Then I added the 2 lines setting some flags, I am not sure the `-no-pointer-arith` is needed but it seems to work with it there
This is the file I was using after edits
```
PROJECT( telldus-core )

CMAKE_MINIMUM_REQUIRED( VERSION 2.6.0 )

CMAKE_POLICY(SET CMP0003 NEW)

SET(PACKAGE_MAJOR_VERSION 2)
SET(PACKAGE_MINOR_VERSION 1)
SET(PACKAGE_PATCH_VERSION 3)
SET(PACKAGE_VERSION "${PACKAGE_MAJOR_VERSION}.${PACKAGE_MINOR_VERSION}.${PACKAGE_PATCH_VERSION}")
SET(PACKAGE_SUBVERSION "beta1")
SET(PACKAGE_SOVERSION 2)

SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

IF (PACKAGE_SUBVERSION)
        SET(DISPLAYED_VERSION "${PACKAGE_VERSION}_${PACKAGE_SUBVERSION}")
ELSE (PACKAGE_SUBVERSION)
        SET(DISPLAYED_VERSION ${PACKAGE_VERSION})
ENDIF(PACKAGE_SUBVERSION)

SET(BUILD_LIBTELLDUS-CORE       TRUE    CACHE BOOL "Build libtelldus-core")

IF (WIN32)
        SET(TDADMIN_DEFAULT FALSE)
ELSEIF(APPLE)
        SET(TDADMIN_DEFAULT FALSE)
ELSE (WIN32)
        SET(TDADMIN_DEFAULT TRUE)
ENDIF (WIN32)

IF (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
        INCLUDE_DIRECTORIES(/usr/local/include)
        LINK_DIRECTORIES(/usr/local/lib)
ENDIF (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")

SET(BUILD_TDTOOL        TRUE                            CACHE BOOL "Build tdtool")
SET(BUILD_TDADMIN       ${TDADMIN_DEFAULT}      CACHE BOOL "Build tdadmin")

SET(GENERATE_MAN        FALSE   CACHE   BOOL "Enable generation of man-files")

ADD_SUBDIRECTORY(common)
ADD_SUBDIRECTORY(service)
ADD_SUBDIRECTORY(client)

IF(BUILD_TDTOOL)
        IF(WIN32)
                ADD_SUBDIRECTORY(3rdparty/openbsd-getopt)
        ENDIF()
        ADD_SUBDIRECTORY(tdtool)
ENDIF(BUILD_TDTOOL)
IF(BUILD_TDADMIN)
        ADD_SUBDIRECTORY(tdadmin)
ENDIF(BUILD_TDADMIN)

ENABLE_TESTING()
ADD_SUBDIRECTORY(tests)

# Remove Doxygen as there was something that was not working, not necessary

#FIND_PACKAGE(Doxygen)

#IF(DOXYGEN_FOUND)
#       SET(DOXY_CONFIG ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
#
#       CONFIGURE_FILE(
#               "${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in"
#               ${DOXY_CONFIG} @ONLY
#       )
#
#       ADD_CUSTOM_TARGET(docs
#               ${DOXYGEN_EXECUTABLE} ${DOXY_CONFIG}
#               DEPENDS ${DOXY_CONFIG}
#               WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
#               COMMENT "Generating doxygen documentation" VERBATIM
#       )
#ENDIF()

# Add some flags, maybe not needed, see change to common/common.h
set(CMAKE_C_FLAGS "-pthread")
set(CMAKE_CXX_FLAGS "-pthread -no-pointer-arith")

```

##### common/common.h
I get compilations errors that indicated that the headerfile pthread.h was missing so I added that at the end of headerfiles 

```
#ifndef TELLDUS_CORE_COMMON_COMMON_H_
#define TELLDUS_CORE_COMMON_COMMON_H_

#ifdef _WINDOWS
#define strcasecmp _stricmp
#define strncasecmp _strnicmp
#include <ole2.h>
#include <windows.h>
#else
#include <unistd.h>
#endif
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#ifdef _WINDOWS
#include <fstream>  // NOLINT(readability/streams)
#endif
#include <string>
#include "common/Strings.h"

# Added this to get compilation to work
#include "pthread.h"

<...>

```

##### service/SettingsConfuse.cpp

**Scary, note that I am not a C++-developer and my C is very rusty. Here is dragons!**

I get some errors because of better checks in the modern compilator. So I needed to change the code in four places.

The error I get was
```
 error: ordered comparison of pointer with integer zero ('cfg_t*' and 'int')
```
Example of code that did not compile (line 45)
```
        if (d->cfg > 0) {
                cfg_free(d->cfg);
        }

```
I changed it to compare with NULL instead as pointer never can be an integer.
```
        if (d->cfg != NULL) {
                cfg_free(d->cfg);
        }

```
The changes was made at line 45, 48, 59 and 71. The changed part of the file is attached below

```
#include <confuse.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <string>
#include "service/Settings.h"
#include "service/config.h"
#include "client/telldus-core.h"
#include "common/Strings.h"
#include "service/Log.h"

class Settings::PrivateData {
public:
        cfg_t *cfg;
        cfg_t *var_cfg;
};

bool readConfig(cfg_t **cfg);
bool readVarConfig(cfg_t **cfg);

const char* CONFIG_FILE = CONFIG_PATH "/tellstick.conf";
const char* VAR_CONFIG_FILE = VAR_CONFIG_PATH "/telldus-core.conf";

/*
* Constructor
*/
Settings::Settings(void) {
        TelldusCore::MutexLocker locker(&mutex);
        d = new PrivateData;
        readConfig(&d->cfg);
        readVarConfig(&d->var_cfg);
}

/*
* Destructor
*/
Settings::~Settings(void) {
        TelldusCore::MutexLocker locker(&mutex);
        if (d->cfg != NULL) {
                cfg_free(d->cfg);
        }
        if (d->var_cfg != NULL) {
                cfg_free(d->var_cfg);
        }
        delete d;
}

/*
* Return a setting
*/
std::wstring Settings::getSetting(const std::wstring &strName) const {
        TelldusCore::MutexLocker locker(&mutex);
        if (d->cfg != NULL) {
                std::string setting(cfg_getstr(d->cfg, TelldusCore::wideToString(strName).c_str()));
                return TelldusCore::charToWstring(setting.c_str());
        }
        return L"";
}

/*
* Return the number of stored devices
*/
int Settings::getNumberOfNodes(Node node) const {
        TelldusCore::MutexLocker locker(&mutex);
        if (d->cfg != NULL) {
                if (node == Device) {
                        return cfg_size(d->cfg, "device");
                } else if (node == Controller) {
                        return cfg_size(d->cfg, "controller");
                }
        }
        return 0;
}

```

### Compiling and installing

Following the standard way of configuring and compiling programs I used the commands below

```
cmake .
make
sudo make install
sudo ldconfig
```

### Setting up telldus-core
#### Adding an service

Create a file in /lib/systemd/system/ in the target host

```
sudo nano /lib/systemd/system/telldus.system
```
Add the content from below and save the file.
**telldus.system**
```
[Unit]
Description=Tellstick service daemon
After=multi-user.target

[Service]
Type=forking
ExecStart=/usr/local/sbin/telldusd

[Install]
WantedBy=multi-user.target

```

#### Enable and start the service
```
sudo systemctl daemon-reload
sudo systemctl enable telldusd.service
sudo systemctl start telldusd.service
```
Check the status of the service with
```
sudo systemctl status telldusd.service
```
```
● telldusd.service - Tellstick service daemon
     Loaded: loaded (/lib/systemd/system/telldusd.service; enabled; preset: enabled)
     Active: active (running) since Fri 2025-08-29 23:36:16 CEST; 1h 36min ago
    Process: 527 ExecStart=/usr/local/sbin/telldusd (code=exited, status=0/SUCCESS)
   Main PID: 528 (telldusd)
      Tasks: 6 (limit: 1568)
        CPU: 6.771s
     CGroup: /system.slice/telldusd.service
             └─528 /usr/local/sbin/telldusd

Aug 29 23:36:16 rp-3B+ systemd[1]: Starting telldusd.service - Tellstick service daemon...
Aug 29 23:36:16 rp-3B+ telldusd[528]: telldusd daemon starting up
Aug 29 23:36:16 rp-3B+ systemd[1]: Started telldusd.service - Tellstick service daemon.
Aug 29 23:36:16 rp-3B+ telldusd[528]: Connecting to TellStick (1781/C31) with serial A7Z2Z7Z6
```
Optional: check the journal. You get the same information from the status command above
```
journalctl _SYSTEMD_UNIT=telldusd.service 
```

#### Add configuration
Add config (tellstick.conf) that is necessary for running the core-tools
```
sudo nano /etc/tellstick.conf
```
```
# Configuration for tellstick / tellstick duo
# Always restart the service after changing the configuration
# sudo service telldusd restart
user = "nobody"
group = "plugdev"
deviceNode = "/dev/tellstick"
ignoreControllerConfirmation = "false"

controller {
  id = 0
  # name = ""
  type = 2
  serial = "XXXXXXXX"
}
```

I created the smallest possible file, I have no devices configurerd as I only uses my Tellstick duo for access to some old sensors. There are examples of how to add devices in the sources below.

Find the serial by running usb-devices or other command like that.
```
usb-devices
```
```
T:  Bus=01 Lev=03 Prnt=06 Port=02 Cnt=01 Dev#=  4 Spd=12  MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 8 #Cfgs=  1
P:  Vendor=1781 ProdID=0c31 Rev=06.00
S:  Manufacturer=Telldus
S:  Product=TellStick Duo
S:  SerialNumber=XXXXXXXX
C:  #Ifs= 1 Cfg#= 1 Atr=80 MxPwr=90mA
I:  If#= 0 Alt= 0 #EPs= 2 Cls=ff(vend.) Sub=ff Prot=ff Driver=usbfs
E:  Ad=02(O) Atr=02(Bulk) MxPS=  64 Ivl=0ms
E:  Ad=81(I) Atr=02(Bulk) MxPS=  64 Ivl=0ms
```

```
sudo chown root:plugdev /etc/tellstick.conf
```
Always restart the service after changing the configuration
```
sudo service telldusd restart
```

## Final checks
I rebooted the Raspberry to check that the service starts after an reboot.

Then I could check that everything works for me.
```
tdtool --list
```
```
Number of devices: 0


SENSORS:

PROTOCOL                MODEL                   ID      TEMP    HUMIDITY        RAIN                    WIND                    LAST UPDATED        
oregon                  F824                    5       21.1°   56%                                                             2025-08-30 01:23:36 
fineoffset              temperaturehumidity     183     21.8°   56%                                                             2025-08-30 01:24:10 
fineoffset              temperaturehumidity     167     21.2°   56%                                                             2025-08-30 01:23:07 
fineoffset              temperature             151     15.8°                                                                   2025-08-30 01:24:29 
fineoffset              temperature             200     21.0°                                                                   2025-08-30 01:23:30 
fineoffset              temperature             168     26.1°                                                                   2025-08-30 01:24:07 
fineoffset              temperaturehumidity     199     21.3°   54%                                                             2025-08-30 01:24:30 
fineoffset              temperature             184     15.8°                                                                   2025-08-30 01:24:15 
oregon                  EA4C                    91      15.3°                                                                   2025-08-30 01:23:59 
oregon                  1A2D                    163     16.3°   79%                                                             2025-08-30 01:23:53 
```

**Now it is time to backup the USB stick**

## Troubleshooting or Tips and Tricks
- Make sure /usr/lib/local is in LD_LIBRARY_PATH

## Sources
- [Raspberry Pi + Tellstick Duo + Nexa = Awsome! How to set it up! (In Swedish)](https://blogg.itslav.nu/?p=875)
- [Information on temperature logging with Raspberry Pi, Telldus](https://lassesunix.wordpress.com/2013/08/10/information-raspberry-pi-telldus-and-temperature-sensing/)
- [R-Pi Tellstick core](https://elinux.org/R-Pi_Tellstick_core)
- [Telldus documentation of tellstick.conf (adding devices)](https://developer.telldus.com/wiki/TellStick_conf)
- [Tellstick on Linux - experience sharing(Telldus Forum)](https://forum.telldus.com/viewtopic.php?t=6464)
- [Tellstick Duo resources (Telldus Forum)](https://forum.telldus.com/viewtopic.php?t=683)
- [TellStick installation - Linux (Telldus developer wiki)](https://developer.telldus.com/wiki/TellStickInstallationSource)
- [telldus-core (github)](https://github.com/rubund/telldus-core)
- [libconfuse (github)](https://github.com/libconfuse/libconfuse)
- My own notes from last time I did this

# Fork?

At the moment I have not forked telldus-core as I am not a C++ developer so I can't provide any support.

# License
This is licensed in the same way as telldus-core using LGPL 2.1.