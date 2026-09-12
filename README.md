# PocketBook 626 replace internal storage
Guide/general information about replacing internal storage in the ebook reader PocketBook 626 and other PocketBook devices.
Since i didn't see a definitive guide that worked, i've decided to make one myself, so others won't have to go through hours of torture like me.
The credits go to users on the mobileread.com forum, mainly _nhedgehog_,_m4mmon_ and others.

>I've separated guides to do certain tasks with the actual methods, so it's shorter

## Sources
- [mobileread thread](https://www.mobileread.com/forums/showthread.php?t=278728)
- [Richard Butrons blog post](https://richard.burtons.org/2016/07/01/changing-the-cid-on-an-sd-card/)
- [Lunator's blog post](http://luator.de/linux/2019/11/23/pocketbook-replace-memory.html)
- [4pda forum thread](https://4pda.to/forum/index.php?showtopic=538665&st=7540)

## About the device

### Hardware
- PocketBook 626, and probably most of the other models, except the very old ones, use a micro SD card for internal storage (don't confuse it with the additional storage, that is also a micro SD card
  - As everyone knows, SD cards don't really live that long, and over time, they often die or become read-only. That means it deletes all the stuff you saved after shutting down the reader

### OS
- The device uses linux with a weird configuration to make our lives harder
- It has 10 partitions that look like this:
  - _p1_ is the _main/user data_ partition where books and apps are stored, it's also the partition that shows up when you connect the reader to a PC
```
Device          Boot   Start      End  Sectors   Size Id Type
/dev/mmcblk0p1       1011712 31116287 30104576  14,4G  b W95 FAT32
/dev/mmcblk0p2  *      73728   139263    65536    32M  6 FAT16
/dev/mmcblk0p3             1  1009664  1009664   493M 85 Linux extended
/dev/mmcblk0p5        139264   172031    32768    16M 83 Linux
/dev/mmcblk0p6        172032   204799    32768    16M 83 Linux
/dev/mmcblk0p7        204800   275455    70656  34,5M 83 Linux
/dev/mmcblk0p8        275456   776191   500736 244,5M 83 Linux
/dev/mmcblk0p9        776192   976895   200704    98M 83 Linux
/dev/mmcblk0p10       976896  1009663    32768    16M 83 Linux
```
- The system checks serial number of the SD card, to check if the card is the original one, other embedded systems also do this (GPS devices, kiosks, etc.)
  - So just cloning the whole image to another SD card won't work, if the CID does not match the original one, the reader just shows a sand clock at boot (or another error)

## What you need
- A computer, i did it on Linux
- Computer with internal SD card reader, or alternatively rooted android device with SD card slot
- Working micro SD card that is the same size or bigger than the original

## Guides

### Read CID (Card Identification Data) of an SD card
- Be aware that in order to correctly interact with an SD card, you need a device with embedded card reader. **USB readers can't read the cid**, since the computer sees the device as unspecified USB mass storage device
- It's easiest to do in Linux - `cat /sys/block/mmcblkX/device/cid` - replace X with the correct number (you can check with `lsblk`)
- On Windows, you'll have to use specialized apps, i don't have experience with that, so you'll have to find something yourself
- It's also possible to do in rooted android - You'll have to identify the correct disk number, most probably it's 1 because 0 is device storage
  - To execute the command either download an Android terminal emulator app or use [adb](https://developer.android.com/tools/adb).
    - **How to do it with adb:** `adb shell`, `su`,`/sys/block/mmcblkX/device/cid`

### Read serial number of an SD card
- Read the serial in linux - `cat /sys/block/mmcblkX/device/serial`
- The SD_prepare app used in [this](#for-older-firmware-versions-edit-monitorapp) method also returns the serial number of card that's in the reader
You can also derive the cards SN from its CID:
- CID look like this `824a544e4361726410c708e2ef00e801`
- Its SN looks like this `0xc708e2ef`
- By comparing the old CID with the new CID, you can derive the new correct serial like this:
```
                   c708e2ef
824a544e4361726410 c708e2ef 00e801
02544d534130384714 1286ddca 00e601
```
- So the other serial will be `0x1286ddca`

### Create and write an image of the disk
- I've used `dd` for simplicity
- There's loads of alternatives for both Linux and Windows (like Win32DiskImager)
Create image - `dd if=/dev/mmcblkX of=/path/to/pocketbook_image.dd bs=512 conv=noerror,sync status=progress`
Write image - `dd if=/path/to/pocketbook_image.dd of=/dev/mmcblkX bs=512 conv=noerror,sync status=progress`

### Make the main partition larger
- Filesystem type is FAT32
- In order to do that, the partition has to be deleted and recreated, so **backup all contents of the partition** so you can later put them into the new partition
- Also, in my case. After booting up the reader with the resized partition, the device went into update mode. After the update, i couldn't open any of the books because of DRM protection. Those books were previously fine, others haven't reported anything like that
- I've used fdisk:
```
 ~ $ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.31.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device          Boot   Start     End Sectors   Size Id Type
/dev/mmcblk0p1       1009664 7747583 6737920   3.2G  b W95 FAT32
/dev/mmcblk0p2  *      73728  139263   65536    32M  6 FAT16
/dev/mmcblk0p3             1 1009664 1009664   493M 85 Linux extended
/dev/mmcblk0p5        139264  172031   32768    16M 83 Linux
/dev/mmcblk0p6        172032  204799   32768    16M 83 Linux
/dev/mmcblk0p7        204800  275455   70656  34.5M 83 Linux
/dev/mmcblk0p8        275456  776191  500736 244.5M 83 Linux
/dev/mmcblk0p9        776192  976895  200704    98M 83 Linux
/dev/mmcblk0p10       976896 1009663   32768    16M 83 Linux

Partition table entries are not in disk order.

Command (m for help): d
Partition number (1-3,5-10, default 10): 1

Partition 1 has been deleted.

Command (m for help): n
Partition type
   p   primary (1 primary, 1 extended, 2 free)
   l   logical (numbered from 5)
Select (default p): p
Partition number (1,4, default 1):
First sector (1009665-31116287, default 1009665): 
Last sector, +sectors or +size{K,M,G,T,P} (1009665-31116287, default 31116287): 

Created a new partition 1 of type 'Linux' and of size 14.4 GiB.

Command (m for help): t
Partition number (1-3,5-10, default 10): 1
Hex code (type L to list all codes): b

Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): p
Disk /dev/mmcblk0: 14,9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device          Boot   Start      End  Sectors   Size Id Type
/dev/mmcblk0p1       1011712 31116287 30104576  14,4G  b W95 FAT32
/dev/mmcblk0p2  *      73728   139263    65536    32M  6 FAT16
/dev/mmcblk0p3             1  1009664  1009664   493M 85 Linux extended
/dev/mmcblk0p5        139264   172031    32768    16M 83 Linux
/dev/mmcblk0p6        172032   204799    32768    16M 83 Linux
/dev/mmcblk0p7        204800   275455    70656  34,5M 83 Linux
/dev/mmcblk0p8        275456   776191   500736 244,5M 83 Linux
/dev/mmcblk0p9        776192   976895   200704    98M 83 Linux
/dev/mmcblk0p10       976896  1009663    32768    16M 83 Linux

Partition table entries are not in disk order.

Command (m for help): w
The partition table has been altered.
Synching disks.


 ~ $ sudo mkdosfs -F 32 -I /dev/mmcblk0p1
```
## Methods
- There are multiple ways, but unfortunately, it varies for different firmware versions
- If any of these methods work and you used bigger card than the original, you can try to resize the main partition [like this](#make-the-main-partition-larger)

### Easiest method: Buy unlocked micro SD card
- This method is obviously firmware independent since you won't be altering the system
- Look up _unlocked CID micro SD_ or _custom CID micro SD_ or something along those lines. They are also sometimes called _coldcards_, mainly chinese sellers will pop up, but i've found a legit seller from Europe - [zelemar.eu](https://zelemar.eu/)
- You can either buy card that will already ship to you with with your custom CID, or the CID can be changed with some software they provide
- When you have the new working card, just [write the image to it](#Create-and-write-an-image-of-the-disk) and you're good to go

### For older firmware versions: Edit monitor.app
This method has been patched for a long time, so it probably won't work

1. Download the SD_prepare.zip from [here](SD_prepare.zip)
2. Extract it on the new microSD card
3. Insert the card into the e-reader using the external card slot
4. Start the device and execute the SD_prepare application (should be shown in the list of applications). This will create two files .sd_original_serial and monitor.app on the card
5. Remove the card and safe the two files somewhere else
6. Open the monitor.app in a hex editor (i used Ghex)
   - Search for this string `/sys/block/mmcblk%c/device/serial` - that's where the monitor.app gets the cards sn
7. Replace this string with /mnt/secure/.sd_original_serial
   - **This string is little shorter**, you can either pad the beggining with couple zeros or make file name longer, e.g. `/mnt/secure/.sd_original_serialll`, but you'll have to rename the file to match it
8. Save the modified file with the name _monitor_patched.app_
9. Mount /dev/mmcblkXp9 - `sudo mount /dev/mmcblkXp9 /some/folder`, and copy the file .sd_original_serial to it
10. Mount /dev/mmcblkXp8 - `sudo mount /dev/mmcblkXp8 /some/folder1`, and copy the file monitor_patched.app to it
11. In the directory in which you mounted partition 8 execute the following commands:

```
chmod a+x monitor_patched.app
mv pocketbook pocketbook_ORIG
ln -s monitor_patched.app pocketbook
```

12. Unmout the two partitions `sudo umount /dev/mmcblkXp8 && sudo mount /dev/mmcblkXp9`
13. Put the new card into the reader and see if it worked

### For newer firmware (2016) versions: Replace .freezestatus file
1. Find out CIDs of both the new and original SD card - [guide](#read-cid-of-an-sd-card)
2. Find out your readers serial number (YT123456..)
   - It's written on the inner side of the back cover
   - It's also written in _Settings->Info_ in the reader
4. Write the original image to the new card - [guide](#create-and-write-an-image-of-the-disk)
5. Mount partition 9: `sudo mount /dev/mmcblkXp9 /some/folder`
6. Download the _freezestatus_ tool from [here](progs.7z)
7. If you're on linux, compile it by running `make` in the _Linux_ folder
8. In the serial folder, execute this `./serial --serial_number <ur serial number> --sd_serial 0x<ur sn>`
9. In the same folder, _.freezestatus_ file was created
10. Overwrite the _.freezestatus_ file in the mounted partition
11. Unmount the partition - `sudo umount /dev/mmcblkXpí`
12. Insert the card into your reader and enjoy
- Optionally, if you got bigger card, you can resize the partition [like this](#make-the-main-partition-larger)

### Changing CID of Samsung Evo+ SD card
This does not work or SD cards manufactured recently, if you have old genuine Evo+ lying around (or no other option), you can try this
- You need rooted Samsung device with micro SD slot
- Credit goes to [this guy](https://github.com/raburton/evoplus_cid)
1. [Find out](#read-CID-Card-Identification-Data-of-an-SD-card) the original CID
2. Use the evoplu_cid tool to try changing CID on another working card - [guide](https://richard.burtons.org/2016/07/01/changing-the-cid-on-an-sd-card/)
4. If the results succeeds, check the new CID
5. Write the original SD card image to the new card [like this](#create-and-write-an-image-of-the-disk) and enjoy
