# Adding a Kvaser CAN Interface (Example: Kvaser U100)

The library only accepts Kvaser USB devices whose USB Product ID is listed in
the module `KvaserCAN_Devices`. Any other device is ignored during enumeration,
even if it is plugged in. The Kvaser U100 (01173-1, Product ID 273 / 0x0111) is
now enabled; the steps below show how it was done, so you can repeat them for
another single-channel interface.

## 1. Find the USB Product ID

Plug in the device and run:

```
ioreg -p IOUSB -l | grep -A10 -i kvaser
```

Note `idVendor` (Kvaser = 2989 / 0xBFD) and `idProduct` (U100 = 273).
Also check the device family: CAN FD interfaces (U100, U100P, Leaf Pro HS v2, ...)
belong to the *Mhydra* family, classic CAN interfaces (Leaf Light v2, ...) to the
*Leaf* family.

## 2. Enable the device option

In `Sources/Driver/KvaserCAN_Devices.h` set the device option to `1`:

```c
#define OPTION_USB_U100_DEVICE                   1 ///< Kvaser U100 (01173-1)
```

## 3. Define the device properties

In the same file, replace the `#error Device properties not defined!` line of
the device's block with its attributes (copy them from a similar, already
working device of the same family — here the U100P):

```c
#if (OPTION_USB_U100_DEVICE != 0)
    #define USB_U100_PRODUCT_ID  273U
    #define USB_U100_DRV_FAMILY  KVASER_USB_MHYDRA_DEVICE_FAMILY
    #define USB_U100_NUM_CHANNELS  1U
    #define USB_U100_CAN_CLOCK   80U
    #define USB_U100_TIMER_FREQ  24U
    #define USB_U100_CAP_CANFD   true
    #define USB_U100_CAP_NONISO  true
    #define USB_U100_CAP_SILENT_MODE  true
    #define USB_U100_CAP_ERROR_FRAME  true
#endif
```

## 4. Add the device to both device lists

In `Sources/Driver/KvaserCAN_Devices.c` the device appears twice. Replace both
`#error Device not supported!` lines for the device:

USB device list (`CANDEV_Devices[]`, used for USB enumeration):

```c
#if (OPTION_USB_U100_DEVICE != 0)
    {KVASER_VENDOR_ID, USB_U100_PRODUCT_ID, USB_U100_NUM_CHANNELS},
#endif
```

Property table (`Kvaser_Devices[]`, used by the driver):

```c
#if (OPTION_USB_U100_DEVICE != 0)
    {USB_U100_PRODUCT_ID, USB_U100_DRV_FAMILY, USB_U100_NUM_CHANNELS, USB_U100_CAN_CLOCK, USB_U100_TIMER_FREQ, USB_U100_CAP_CANFD, USB_U100_CAP_NONISO, USB_U100_CAP_SILENT_MODE, USB_U100_CAP_ERROR_FRAME},
#endif
```

## 5. Rebuild and reinstall

```
make clean
make all
sudo make install
```

Then rebuild/relink your application against the new library (or copy the new
`libKvaserCAN.dylib` / `libUVCANKVL.dylib` next to it) and check that the device
is found, e.g. with the `can_moni` utility:

```
can_moni --list-boards    # the device must be listed as supported
can_moni --test-boards    # with the device plugged in: it must be shown as available
```

Notes:

- Only single-channel devices are fully supported at the moment.
- If the device does not show up, re-check the Product ID and that the option
  is `1`; a device with option `1` but missing properties stops the build with
  `#error`, which is intended.
