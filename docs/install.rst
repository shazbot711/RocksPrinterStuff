Installation Guide
==================

This document will guide you through the process of setting up mariner with
your printer: from the hardware setup, installing the mariner software, and
setting up Linux to share files with the printer and use the serial port.

.. note::
   This software is provided "as is", without warranty of any kind. There is no
   guarantee this will work on your printer, nor that it won't damage it.  Use
   at your own risk.

Requirements
------------

Before you get started, make sure you have the following:

* Raspberry Pi Zero W
* One of the `supported 3D Printers <https://github.com/luizribeiro/mariner#supported-printers>`_
* Micro USB to USB-A cable to connect between Pi and printer mainboard.
* Power Supply for the Pi, or a 12V to 5V converter to power it from
  printer's power supply.

Hardware setup
--------------

These are high level instructions for the hardware setup:

1. Setup a Raspberry Pi Zero W with ssh access over WiFi.
2. Wire it to the serial port on the ChiTu mainboard. Do not connect the 5V
   line.
3. Connect the USB OTG port on the Pi to the USB port of the mainboard. Do
   not connect the 5V line. You can put some tape on the connector to
   isolate the 5V line of the USB cable.
4. Connect Pi's USB PWR port to a power supply. You can also use a 12V to 5V
   converter from the printer's power supply.

For more detailed instructions for the Elegoo Mars Pro, refer to `this blog post
<https://l9o.dev/posts/controlling-an-elegoo-mars-pro-remotely/>`_. The setup
for other printers should be almost identical.

Installing package
------------------

First, enable the repository:

.. code-block:: bash

   $ curl -sL gpg.l9o.dev | sudo apt-key add -
   $ echo "deb https://ppa.l9o.dev/raspbian ./" | sudo tee /etc/apt/sources.list.d/l9o.list
   $ sudo apt update

Then install mariner:

.. code-block:: bash

   $ sudo apt install mariner3d

USB Gadget Setup
----------------

In order to make the printer see the files uploaded to mariner, we need to
setup the `USB Gadget driver <https://www.kernel.org/doc/html/latest/driver-api/usb/gadget.html>`_
as a Mass Storage device. This section will guide you through that
process.

Enable USB driver for gadget modules by adding this line to
``/boot/config.txt``:

.. code-block:: bash

   dtoverlay=dwc2

Enable the dwc2 kernel module, by adding this to your ``/boot/cmdline.txt``
just after ``rootwait``:

.. code-block:: bash

   modules-load=dwc2

Setup a container file for storing uploaded files:

.. code-block:: bash

   $ sudo dd bs=1M if=/dev/zero of=/piusb.bin count=2048
   $ sudo mkdosfs /piusb.bin -F 32 -I

Create the mount point for the container file:

.. code-block:: bash

   $ sudo mkdir -p /mnt/usb_share

Add the following line to your ``/etc/fstab`` so the container file gets
mounted on boot::

   /piusb.bin /mnt/usb_share vfat users,gid=mariner,umask=002 0 2

Finally, make ``/etc/rc.local`` load the ``g_mass_storage`` module by adding
this to it:

.. code-block:: sh

   #!/bin/sh -e

   modprobe g_mass_storage file=/piusb.bin stall=0 ro=1

   exit 0

Once you restart the pi (or potentially run ``sudo mount -a``), the printer
should start seeing the contents of ``/mnt/usb_share``.

Setting up the serial port
--------------------------

First, enable UART by adding this to ``/boot/config.txt``::

   enable_uart=1

In order for the Pi to communicate with the printer's mainboard over
serial, you also need to disable the Pi's console over the serial port:

.. code-block:: bash

   $ sudo systemctl stop serial-getty@ttyS0
   $ sudo systemctl disable serial-getty@ttyS0

Lastly, remove the console from ``cmdline.txt`` by removing this from it::

   console=serial0,115200

Wrapping up
-----------

Reboot the Pi and you should be all set. Again these are rough
instructions for now :)

You can check that the ``mariner3d`` service is running with:

.. code-block:: bash

   $ sudo systemctl status mariner3d

If it is, you should be able to access it by opening
``http://<pi ip address>:5000/`` on your browser.