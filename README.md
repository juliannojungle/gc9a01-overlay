# GC9A01 FBTFT overlay

This is an overlay for the `fb_ili9340` graphics driver from [NoTro FBTFT](https://github.com/notro/fbtft/wiki/FBTFT-RPI-overlays), to use with LCD displays that has the [Galaxycore's GC9A01 single chip driver](GC9A01A.pdf). It allows to easily setup (in just 3 super easy steps!) said displays to be used on newer Raspberry Pi OS releases that already includes `fbtft` on it's kernel.

## Step #1: Wiring!

The display should be connected to the Raspberry Pi on the first SPI channel (`spi0`) [pins](https://pinout.xyz), as follows:

<table>
    <thead>
        <tr>
            <td>LCD</td><td>GPIO</td><td>Raspberry Pi physical pin</td>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>VCC</td><td>3.3V</td><td>1</td>
        </tr>
        <tr>
            <td>GND</td><td>GND</td><td>6</td>
        </tr>
        <tr>
            <td>DIN</td><td>10 (spi0 MOSI)</td><td>19</td>
        </tr>
        <tr>
            <td>CLK</td><td>11 (spi0 SCLK)</td><td>23</td>
        </tr>
        <tr>
            <td>CS</td><td>8 (spi0 CE0)</td><td>24</td>
        </tr>
        <tr>
            <td>DC</td><td>25</td><td>22</td>
        </tr>
        <tr>
            <td>RST</td><td>27</td><td>13</td>
        </tr>
        <tr>
            <td>BL</td><td>18 (pcm clock)</td><td>12</td>
        </tr>
    </tbody>
</table>

## Step #2: Setup!

1. Locate your sdcard boot partition. If you are on 'Windows', that should be the partition where the sdcard was mounted (e.g. `E:/`). On 'Raspberry Pi OS' that should be `/boot`;

2. Download the [gc9a01.dtbo](https://github.com/juliannojungle/gc9a01-overlay/releases/download/v1.0.0/gc9a01.dtbo) file to the `overlays` directory in boot partition (e.g. `E:/overlays` on 'Windows' or `/boot/overlays` on 'Raspberry Pi OS');

3. Edit the `config.txt` file on the boot partition and append the following line to the end of the file:

```
dtoverlay=gc9a01
```
The line above will attach GC9A01 LCD driver to `/dev/fb1` framebuffer over `spi0` spi pins and initialize the LCD.

That's it. Put the sdcard on the Raspberry Pi and boot (if you did the above steps right inside from 'Raspberry Pi OS', just reboot with `sudo reboot`).

After power up, open a terminal and verify that the device was properly mounted:

```
ls /dev/fb*
```

- this should list both `fb0` and `fb1`.

## Step #3: Get some image!

Since this overlay is just an extension of the device driver, it only attaches and initiates the LCD device on the `fb1` framebuffer (it's like turning on the TV without any cable or antenna input). In order to actually see something on the display, you need something sending image to it.
What users tipically do is just mirror the HDMI output (displayed on `fb0`) on the LCD (displayed on `fb1`). For this task there are many tools available and we'll help you to setup one of them bellow. If you are a developer, another way to show stuff on the display would be your application directly write on `fb1` framebuffer, but that won't be covered here.

### Mirroring HDMI on LCD: Rpi-fbcp

[Raspberry Pi Framebuffer Copy](https://github.com/tasanakorn/rpi-fbcp) is a tool that copies the primary framebuffer (`fb0`) to a secondary one (`fb1`).

Run the following commands to download, build and install:

```
cd ~
git clone https://github.com/tasanakorn/rpi-fbcp
cd rpi-fbcp/
mkdir build
cd build/
cmake ..
make
sudo install fbcp /usr/local/bin/fbcp
```

To make it run on boot, edit the following file:

```
sudo vi /etc/rc.local
```

Add `fbcp&` on the line right before `exit 0`. The `&` will make it run on background, without hanging the boot process:

```
fbcp&
exit 0
```
Reboot the Raspberry Pi and you'll start seeing the image from HDMI mirrored on the LCD.

# Extra setup (optional)

## Overlay parameters

The overlay support some optional parameters that allow changes in the default behavior and affects only the LCD display. They are key=value pairs, comma separated in no predefined order, as follow:

```
dtoverlay=gc9a01,speed=40000000,rotate=0,width=240,height=240,fps=50,debug=0
```

- `speed`: max spi frequency to be used
- `rotate`: image rotation (in degrees: 0, 90, 180, 270)
- `width`: width of the display
- `height`: height of the display
- `fps`: max fps to be used
- `debug`: debug level to be logged on boot process

## Additional image orientation and resolution

Since `fbcp` is making a plain copy from HDMI to LCD, screen resolution may affect the final result. Additional settings can be added on the `config.txt` in order to adjust the resulting image to your needs. The full set of options can be checked at [/boot/overlays/README](https://github.com/raspberrypi/linux/blob/rpi-5.10.y/arch/arm/boot/dts/overlays/README).

Note that the following settings will be applied both to the HDMI and the LCD.

```
dtoverlay=gc9a01
hdmi_force_hotplug=1
hdmi_cvt=240 240 60 1 0 0 0
hdmi_group=2
hdmi_mode=87
hdmi_drive=2
display_rotate=2
```

- `hdmi_force_hotplug`: force HDMI output rather than DVI
- `hdmi_cvt`: adjusts de resolution, framerate and more. Format: \<width\> \<height\> \<framerate\> \<aspect\> \<margins\> \<interlace\>
- `hdmi_group`: set DMT group (Display Monitor Timings: the standard typically used by monitors)
- `hdmi_mode`: set DMT mode
- `hdmi_drive`: force a HDMI mode rather than DVI
- `display_rotate`: rotate screen 180 degrees

The `display_rotate` setting allows to rotate or flip the screen orientation to fit your needs. The default value is `0`, possible values are:

- `0` no rotation
- `1` rotate 90 degrees clockwise
- `2` rotate 180 degrees clockwise
- `3` rotate 270 degrees clockwise
- `0x10000` horizontal flip
- `0x20000` vertical flip

This setting is a bitmask. So you can both flip and rotate the display at the same time. Example:

- `0x10001` both do a horizontal flip and rotate 90 degrees clockwise (`0x10000` + `1`).
- `0x20003` both do a vertical flip and rotate 270 degrees clockwise (`0x20000` + `3`).

<br>

# Development

## Building and testing

Clone the repository:

```
cd ~
git clone https://github.com/juliannojungle/gc9a01-overlay
cd gc9a01-overlay
```

Build overlay:

```
dtc -W no-unit_address_vs_reg -@ -I dts -O dtb -o gc9a01.dtbo gc9a01-overlay.dts
```

Load overlay:

```
sudo dtoverlay -v -d . gc9a01.dtbo
```

Test loaded overlay:

```
ls /dev/fb*
```

- should list both `fb0` and `fb1`

Unload overlay:

```
sudo dtoverlay -r gc9a01
```

Check for info on boot process:

```
dmesg | grep graphics
```

- should list the loaded driver info

<br>

---
<sup>[@juliannojungle](https://github.com/juliannojungle), 2022</sup>