Title: Provisioning Titan Security Keys From Linux
Date: 2018-10-02 19:30
Tags: linux, security

Google has released their own multifactor authentication hardware tokens: [Titan Security Keys](https://cloud.google.com/titan-security-key/). Configuring your Google account to require the hardware keys is typically pretty [straightforward](https://support.google.com/accounts/answer/6103523) but they must be added from a computer. When I was trying from my Linux system the process would always error out, just stating "something went wrong". I eventually found [directions](https://support.google.com/titansecuritykey/answer/9148044?hl=en) from Google that look pretty straight forward, but they did not resolve my issue. Turns out a few tweaks were required.

# Official Method

The initial directions as listed are as follows:

1. Create a file called `/etc/udev/rules.d/titan-key.rules`
2. Populate the file with the following:
```
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="18d1", ATTRS{idProduct}=="5026", TAG+="uaccess"
```
3. Save
4. Reboot

# Troubleshooting

Even after following the above, I was getting the same error. Looking more closely at the rule as configured gave me some clues as to why. Essentially, the rule says that when a device is inserted that has a vendorID of 18d1 and a productID of 5026, add a uaccess tag. This allows Chrome to communicate directly with the USB device. That is required by the key since [FIDO U2F](https://en.wikipedia.org/wiki/Universal_2nd_Factor), the protocol in use actually utizilizes the URL when creating the challenge–response authentication request. Plugging a key in and checking `dmesg` reveals the issue:
```
[ 5866.626801] usb 1-2: new full-speed USB device number 16 using xhci_hcd
[ 5866.775955] usb 1-2: New USB device found, idVendor=096e, idProduct=0858, bcdDevice=44.00
[ 5866.775962] usb 1-2: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[ 5866.775966] usb 1-2: Product: U2F
[ 5866.775969] usb 1-2: Manufacturer: FT
```
Checking both keys from the set confirmed that neither the vendor or product matches Google's documentation. After looking at a few different articles I found you can configure udev rules to match multiple options in a given field. Additionally, my version of udev did not seem to recognize rules unless the file name was refixed with a number.

# Fixed Directions

1. Create a file called `/etc/udev/rules.d/70-titan-key.rules`
2. Populate with the following:
```
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="18d1|096e", ATTRS{idProduct}=="5026|0858|085b", TAG+="uaccess"
```
3. Save
4. Run `sudo udevadm control --reload-rules`

Provision the key should now work from your Linux machine!

These directions were tested on openSUSE Tumbleweed with udev version 237.