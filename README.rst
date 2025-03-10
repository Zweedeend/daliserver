0. Contents
-----------

1. About
2. Installation
3. Running
4. Communication Protocol
5. Copyright
6. References

1. About
--------

DALI (Digital Addressable Lighting Interface) is bus-based lamp control sytem,
standardized by the DALI working group (DALI-AG), a subsidiary of the German
ZVEI (Zentralverband Elektrotechnik- und Elektronikindustrie e.V.).
It allows a wide range of lighting control applications, such as switching
lamps in groups, automatic leveling in response to daylight intensity,
and integration into home automation systems.

daliserver is a command multiplexing server for the Tridonic DALI USB adapter,
allowing access to a DALI bus from any USB-equipped computer that supports
libusb. Note however that so far, only Linux and Mac OS X have been tested.

daliserver is copyright (c) 2011 by its authors (see below).
Licensing terms can be found in the file LICENSE that you should have received
with the program or source code.
The source code is hosted on https://github.com/onitake/daliserver

2. Installation
---------------

If you received daliserver in binary form, you can skip the next section and go
straight to "Running".

To compile daliserver, you need GNU autotools, a C99 compliant C compiler,
libusb 1.0+, pkg-config and associated development packages.
For Debian based Linux operating systems, package build scripts are provided.
If you want to get the source code straight from its repository, you also need
to install git.

2.1 Debian and Ubuntu
---------------------

If you use Debian Linux or a Debian derivate such as Ubuntu, the following
will generate a Debian package for you:

$ sudo apt-get install git dpkg-dev debhelper autotools-dev autoconf pkg-config libusb-1.0-0-dev

$ tar xvfz daliserver-<version>.tar.gz
$ cd daliserver-<version>
or
$ git clone git://github.com/onitake/daliserver.git
$ cd daliserver

$ autoreconf -i
$ dpkg-buildpackage

$ cd ..
$ ls -l daliserver*.deb

If everything went smoothly, you should see a newly generated package there.
This can be installed with:

$ sudo dpkg -i daliserver_<version>_<architecture>.deb

daliserver will be started during boot afterwards and should already be
running if it can find the USB adapter.

2.2 Other Linux distributions
-----------------------------

If you don't use Debian, you can also build manually.
Install the appropriate packages using your system's package manager, then:

$ tar xvfz daliserver-<version>.tar.gz
$ cd daliserver-<version>
or
$ git clone git://github.com/onitake/daliserver.git
$ cd daliserver

$ autoreconf -i
$ ./configure
$ make

And install to /usr/local with

$ sudo make install

If you want to set up daliserver to run during boot, have a look at
debian/init.d, it should provide hints on how to run daliserver in daemon
mode and set log and run files accordingly.

2.3 Mac OS X
------------

On Mac OS X, you will need to install both Apple developer tools (Xcode with
command line utilities) and pkg-config, libusb-1.0, automake and autoconf.
There are different ways to achieve this, the easiest is probably with the
help of the Homebrew utility.

Xcode can be obtained from the Mac App Store or the Apple Developer Site.

Next, follow the instructions on http://brew.sh/ to install Homebrew, then
install the required packages from a Terminal prompt:

brew install pkg-config libusb autoconf automake

If this step throws any errors during the link step, you might need to clean
up conflicting files or find a way around them. This is outside the scope of
this documentation.

Obtaining and compiling daliserver works like on Linux:

$ tar xvfz daliserver-<version>.tar.gz
$ cd daliserver-<version>
or
$ git clone git://github.com/onitake/daliserver.git
$ cd daliserver

$ autoreconf -i
$ ./configure
$ make

And install to /usr/local with

$ sudo make install

2.4 Windows
-----------

(TODO, untested)

Obtain cygwin from http://cygwin.com/ and install required packages:
gcc libusb pkg-config automake autoconf git

Then, open a command line prompt and checkout and compile daliserver like
on Linux.

3. Running
----------

If you installed daliserver from a package, no further action is required.
In case you didn't attach your DALI USB adapter before installation, you might
have to restart the service. Consult the log file in /var/log/daliserver.log
for errors and diagnostics. You can also simply reboot after attaching your
adapter and fixing permissions (if necessary, see below).

If you don't want to have daliserver running at boot time, or if you use a
different operating system, you may also run daliserver directly. It will
not fork into background by default, but print info messages to the console
instead.

Certain operating systems require special procedures to allow access to
the DALI USB adapter. Tridonic chose to implement a device descriptor that
registers it as a human interface device (HID), so usermode access would be
possible on Microsoft Windows without a kernel mode driver.
On most other modern operating systems, HID devices are normally claimed by a
generic driver that handles human input. Since the adapter is not
actually a HID, special preparations are required on these systems.
On Linux, the generic HID driver supports a special command to detach it from
the device and allow a usermode program access. daliserver sends this command
during startup. On Mac OS X, such a function is not available, but there exists
another way to keep the driver from claiming the adapter. A dummy kernel
extension called DALIUSB.kext is shipped with the source code. Copy it to
/System/Library/Extensions and reboot your system. During the next
start, DALI USB will be registered with the generic USB driver instead of the
HID driver, allowing usermode access from daliserver (or other software).
It might also be necessary to change the device node permissions on certain
Linux variants, as they are normally restricted to super user access. If this
happens, daliserver will print out an error during startup. You may either run
it with superuser privileges (not recommended) or change the permissions of the
device. Read and write permissions are required. It is also possible to automate
this process with udev.

The perl/ subdirectory contains a number of example Perl scripts.
usbdali.pm is a Perl module that can handle construction of DALI messages and
communication with the daliserver. Some examples are provided too:
alloff.pl - Sends a turn off broadcast message
allon.pl - Sends a turn on broadcast message, but no light level (lamps might
still be at 0 after sending this)
allset.pl - Sends a lamp level broadcast message and displays the response
lampoff.pl - Sends a turn off command to a specific lamp
lampset.pl - Sends a level to a specific lamp

4. Communication protocol
-------------------------

To communicate with daliserver, you need to connect to it first.
If you didn't specify any options, it listens on TCP 127.0.0.1:55825

Requests have the following format:

```
  version:uint8_t (protocol version)
  type:uint8_t (request type)
  address:uint8_t (device address)
  command:uint8_t (device command)
```

Responses are defined as:

```
  version:uint8_t (protocol version)
  status:uint8_t (status code)
  response:uint8_t (response value)
  padding:uint8_t (must be 0)
```

Received out of band messages:

```
  version:uint8_t (protocol version)
  status:uint8_t (status code)
  address:uint8_t (device address)
  command:uint8_t (device command)
```

The protocol version is currently 2. All frames are 4 bytes long.

type must be 0, denoting a "Send DALI command" request.

status can have one of the following values:

```
  0:   Transfer successful, no response
  1:   Transfer successful, response received
  2:   Broadcast message received
  255: Transfer error
```

address and command are the DALI device address and command, respectively.

response is the value received from the device, if available, or 0 otherwise.

Successful transfers, responses and broadcast messages can be differentiated
by the value of the status code.

eDALI commands aren't supported for now.

5. Copyright
------------

daliserver is (c) copyright 2010-2015 by
Gregor Riepl <onitake@gmail.com>
Johannes Wüthrich <johannes@deragent.net>
and others.

All rights reserved.
See the AUTHORS file for a list of contributors.
See the LICENSE file for the license terms.

autoconf macros in the m4/ subdirectory are governed by a different license,
see m4/LICENSE for details.

6. References
-------------

http://www.dali-ag.org - DALI working group homepage
IEC 62386 - The official standard (requires fee)
http://www.tridonic.com/com/en/2192.asp - Tridonic masterCONFIGURATOR software
for Microsoft Windows, was used to reverse engineer the communication protocol
with DALI USB and to test out DALI commands
http://www.siwawi.arubi.uni-kl.de/avr_projects/dali/index.html
AVR microcontroller DALI implementation, contains lots of useful links
