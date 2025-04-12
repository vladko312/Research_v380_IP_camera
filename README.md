# v380 IP camera research

![Report version](https://img.shields.io/badge/Report_version-1.0-blue)
![Last modified](https://img.shields.io/badge/Last_modified-13.04.2025-blue)

![CVE-2025-25983](https://img.shields.io/badge/CVE--2025--25983-3.4-green)
![CVE-2025-25984](https://img.shields.io/badge/CVE--2025--25984-6.8-yellow)
![CVE-2025-25985](https://img.shields.io/badge/CVE--2025--25985-3.2-green)

I just decided to learn the basics of IoT hacking, so the camera I used became the obvious target.
The research is not over yet, as I have more ideas to check, while my free time is extremely limited.
I already had some tools and watched some videos, so I decided to look for some simple vulnerabilities first.

> This writeup focuses more on my journey in IoT hacking, while other files in this repository provide more detailed reports of my findings.

- [My tools](#tools)
- [Target info](#target)
- [Initial access](#initial-access)
- [CVE-2025-25983](#qr-based-device-sharing-credentials-leak-cve-2025-25983): QR-based device sharing credentials leak
- [CVE-2025-25984](#root-shell-accessible-over-uart-with-hardcoded-password-cve-2025-25984): Root shell accessible over UART with hardcoded password
- [CVE-2025-25985](#plaintext-credentials-in-the-filesystem-cve-2025-25985): Plaintext credentials in the filesystem
- [Further research](#further-research)

## Tools

Here are some tools I used during my research in order of relevance:

- [FT232RL USB-UART converter](#usb-uart-converter): ~2$
- [Some dupont / arduino wires](#dupont--arduino-wires): ~1$
- [CH341A USB programmer + clip](#ch341a-programmer): ~4$
- [Multimeter DT-182](#multimeter): ~7.5$ (can be replaced with any cheaper one)
- [USB OTG adapter](#usb-otg-adapter): ~0.5$ (optional)
- [Old broken cables](#old-broken-cables): ~0$ (optional)

Yes, that's everything I've used during the research.
All of these tools are quite useful even outside of IoT hacking, so I had most of them already.
The tools can be used for other (much cheaper) IoT targets.

Also, I paid ~2.5$ at a local phone repair shop to solder some thin wires.
I have a soldering station, but I'm not really good at precise soldering yet.

### USB-UART converter

UART is a common debug interface for Linux-based IoT devices.
It can give root access directly or through some tricks.
USB-UART converter is the main tool I used to connect to the camera.

I picked FT232RL-based converter, as it allowed to manually set one of the 3 common UART voltages.
Many converters claim to be compatible with multiple voltages, but only use one voltage for data transmit.

Also, my converter has hardware flow control pins.
Those are rarely needed, but it is nice to have them just in case...

![USB-UART](/images/USB-UART.png)

### Dupont / arduino wires

Useful to connect things without soldering.

![Dupont wires](/images/dupont.png)

### CH341A programmer

This cheap USP programmer can read and write the most common type of memory chips.
It is useful for IoT firmware research and chip data backup.
Outside of IoT hacking, it can be used to restore UEFI/BIOS on motherboards.

The clip can be used to read the chip on the board without desoldering.

![CH341A programmer](/images/programmer.png)

### Multimeter

Not an expensive one, but not the cheapest one either.
It uses an uncommon battery standard, but has sound indication in continuity mode.
You can use any multimeter you find, as it doesn't really matter.

![Multimeter](/images/multimeter.png)

### USB OTG adapter

This adapter allows you to connect USB devices to your phone.
It allowed me to use my USB-UART adapter with my phone.
Not really needed for IoT hacking, but quite useful anyway.

![USB OTG adapter](/images/USB%20OTG.png)

### Old broken cables

Old cables can be used to get connectors and wires for soldering.
I used a connector from an old battery to conveniently access UART even when the camera is assembled.
Also, thin color-coded wires from an old USB charging cable, as they were perfect to get all three wires through a small hole in the lid.
Again, not really needed for IoT hacking, just a matter of convenience.

## Target

My research focuses on a rather obscure device made in China, so I had a hard time figuring out it's actual name.
Here are some names I encountered:

- model_id: HsAKPIQp
- hwname: V380E6_C1
- Sold as: XM-200
- App used to connect: v380 / v380 Pro
- Possible manufacturer: Macro-video Technologies Co.,Ltd

Specs:

- CPU: Anyka AK3918EV300L
- Linux 4.4.282
- Flash: 8 MB (winbond 25Q64JVSIQ)
- WiFi module: BL-M8188FU3
- SD card: up to 128GB
- Price: ~45$

Some images:

![Front](/images/full%20front.png)

![Back](/images/back.png)

## Initial access

There are some methods of accessing v380 cameras using files on an SD card, but they didn't work for my camera.
I will later look into some files on my camera to see if there are any methods working for my model.

The lid with the board can be unscrewed from the back of the camera.
I used USB cable to hold the camera, so it would be on the pictures even though the camera is not powered.

My first idea was to look for the UART port.
Sometimes, the port is clearly marked as UART or the pins are labeled as TX/TXD and RX/RXD.
On the bottom side of the device, there are marked test pads:

![UART marked pins](/images/bottom.jpg)

Also, there is the ground pad used for UART and a Winbond memory chip for later.
I used the multimeter to determine that the device is using 3.3v signals.
Transmit pin of the device should be connected to the receive pin of the adapter and vice versa:

- TXD <—> RX
- RXD <—> TX
- GND <—> GND

And when I powered on the camera, I got nothing...
Seems like this is a debug port that is disabled on production models.

So, I started looking for a different way to get UART.
There were no other marked UART pins or rows of 3/4 test pads, so I checked test pads near CPU on the front.
I connected GND to the shielding of the USB port and used RX pin of my adapter to look for UART TX on the device.

Near the CPU, there were two test pads, one of which turned out to be TX.
I checked some common baud rates and got meaningful output with 115200.
I tried to connect the other pin to the TX on my adapter, and it worked.

I went to a local phone repair shop to solder some wires for convenience:

- White to GND (on the USB shielding)
- Red to RX on the device
- Green to TX on the device

![Wires soldered to UART pads](/images/top.jpg)

I added a connector from an old battery, so now I can access UART while the camera is assembled and installed:

![Camera assembled with the connector for UART](/images/assembled.jpg)

## QR-based device sharing credentials leak: CVE-2025-25983

[Technical details](/CVE-2025-25983.md)

While I was away from home, I decided to check the features of the `v380 Pro` app used to connect to the camera.
Another researcher discovered, that sharing a camera to a different account actually shares plaintext login and password ([CVE-2023-33741](https://nvd.nist.gov/vuln/detail/CVE-2023-33741)).
I did not create an account, so I decided to check the QR-based sharing instead.

In the QR code, credentials were not exposed in plaintext.
Instead, the QR code contained plaintext device id, some key and an encrypted message.
In Chinese IoT devices, security through obscurity is very common.
Companies use encryption to make things harder to copy, not to protect customer data.

So, what can 16 bytes long `"akey"` be used for?
I decided to try AES and got two base64 encoded strings with my login and password in them.
Sharing the key alongside the encrypted data completely removes any benefits of the encryption.

I created a simple Python3 [PoC](/CVE-2025-25983_PoC.py) to decode the contents of the QR code, but it can also be done with online tools like CyberChef:

```
https://gchq.github.io/CyberChef/#recipe=Parse_QR_Code(false)Regular_expression('User%20defined','.*%5C%5C%5E(.*)',true,true,false,false,false,false,'List%20capture%20groups')From_Base64('A-Za-z0-9%2B/%3D',true,true)JPath_expression('akey,msg,did',':')Register('%22(.*)%22:%22(.*)%22:(.*)',false,false,false)Regular_expression('User%20defined','%22.*%22:%22(.*)%22',true,true,false,false,false,false,'List%20capture%20groups')AES_Decrypt(%7B'option':'Base64','string':'$R0'%7D,%7B'option':'Hex','string':'0000000000000000000000000000000'%7D,'ECB','Hex','Raw',%7B'option':'Hex','string':''%7D,%7B'option':'Hex','string':''%7D)Find_/_Replace(%7B'option':'Regex','string':'%5E(.*)$'%7D,'$R0%23$1',true,false,true,false)Fork('%23',':',false)From_Base64('A-Za-z0-9%2B/%3D',true,false)Merge(true)Find_/_Replace(%7B'option':'Regex','string':'%5E(.*)$'%7D,'$R2:$1',true,false,true,false)
```

The app warns the user that the QR code gives control over the device, so it shouldn't be shared.
However, the app does not disclose the fact that the plaintext credentials are shared.
Moreover, there is no mention of ways to revoke access.

## Root shell accessible over UART with hardcoded password: CVE-2025-25984

[Technical details](/CVE-2025-25984.md)

When I got some free time at home, I decided to try accessing camera over UART.
After some initial boot logs, the camera requests login and password to get into the shell.

Common credentials did not work, so I decided to use my programmer to get the contents of the Winbond memory chip on the bottom of the board.
I dumped the contents twice to verify data integrity.

After that, I used `binwalk` to extract files from squashfs and jffs2 partitions of the flash dump.
In `/etc/shadow` on the squashfs partition I found the hash of the user `root`:

```
root:$1$abc123lk$QG3JmuRFzSVpv99ers92a/
```

Googling the hash did not give me any results.
I found one mention of the hash on github, but the modders just changed it in the firmware to their one.
Cracking the hash using `hashcat` also wasn't successful.

So, I tried to get the shell in another way, using the bootloader environment variables.
There was no way to access U-Boot shell normally, so I decided to use a trick.

When the bootloader is accessing the Linux files, it is possible to glitch the chip to cause error and get a U-Boot shell.
To do that, I tried shorting pins 2 (Data Output) and 4 (Ground) of the Winbond chip at the right time.

I got into the U-Boot shell multiple times, but was unable to change the envs due to the length limitations.
At some point, I missed and shorted the wrong pins, causing data corruption on the chip.
I erased the chip using my programmer and restored the flash dump to fix the camera.

After that, I remembered seeing some password in a file from a custom web server for that camera.
Turns out, the password was already known by camera modders, but I just forgot about that by the time I got UART.

The password for `root` is `gzhongshi`, and the password is hardcoded in the read-only squashfs partition.

```
$1$abc123lk$QG3JmuRFzSVpv99ers92a/:gzhongshi
```

## Plaintext credentials in the filesystem: CVE-2025-25985

[Technical details](/CVE-2025-25985.md)

After I got root shell access, I decided to look into the contents of the files.
Most of the files are located in a squashfs filesystem, which is read-only by design.
I focused on the files in JFFS2 filesystem instead.

In some configs I found plaintext credentials:

- `/mnt/mtd/mvconf/user_info.ini`: Camera user login and password
- `/mnt/mtd/mvconf/wifi.ini`: Wi-Fi SSID and password

The files could be accessed from the root shell of the device or from the dump of the flash chip.

## Further research

I have plans to continue this research, mainly to look for software and cloud vulnerabilities that can be exploited from the network.