BOOTBOOT Bootable Disk Image Creator
====================================

See [BOOTBOOT Protocol](https://gitlab.com/bztsrc/bootboot) for common details.

This is an all-in-one, multiplatform, dependency-free disk image creator tool. You pass a disk configuration to it in a very
flexible JSON, and it generates ESP FAT boot partition with the required loader files, GPT partitioning table, PMBR, etc. It
also creates an initrd from a directory (currently `cpio`, `tar`, `jamesm` (James Molloy's initrd) and `FS/Z` supported, but
the code is written in a way that it is easily expandable).

The generated image was tested with fdisk, and with the verify function of gdisk. The FAT partition was tested with fsck.vfat
and with TianoCore UEFI firmware and on Raspberry Pi.

Operating Modes
---------------

```
$ ./mkbootimg
BOOTBOOT mkbootimg utility - bztsrc@gitlab
 BOOTBOOT Copyright (c) bzt MIT https://gitlab.com/bztsrc/bootboot
 deflate 1.2.11 Copyright 1995-2017 Jean-loup Gailly and Mark Adler
 Raspbery Pi Firmware Copyright (c) Broadcom Corp, Raspberry Pi (Trading) Ltd

Validates ELF or PE executables for being BOOTBOOT compatible, otherwise
creates a bootable hybrid image for your hobby OS or Option ROM image.

Usage:
  ./mkbootimg check <kernel elf / pe>
  ./mkbootimg <configuration json> initrd.rom
  ./mkbootimg <configuration json> bootpart.bin
  ./mkbootimg <configuration json> <output disk image name>

Examples:
  ./mkbootimg check mykernel/mykernel.x86_64.elf
  ./mkbootimg myos.json initrd.rom
  ./mkbootimg myos.json bootpart.bin
  ./mkbootimg myos.json myos.img
```

If the first argument is `check`, then it's followed by a kernel filename. The utility will check the executable for
BOOTBOOT compliance, and it will report all errors and if passed, which BOOTBOOT Protocol level it conforms to.

Otherwise the first argument is the configuration JSON file. If the second argument is `initrd.rom`, then it will generate
a BIOS Option ROM image from the initrd directory. If that is `bootpart.bin`, then it saves the boot partition image
(and only the partition image). Every other filename will make it generate a whole disk image with GPT.

The tool is multilingual. It will detect your operating system's language and if it has a dictionary for it, it will use that.
You can override the autodetection from the command line by using the `-l <lang>` flag as the first argument (available for
all operating modes). Language is given in two characters long code and fallbacks to `en`.

Configuration
-------------

The JSON is simple and flexible, accepts many variations. At the top level, you can define the output disk parameters.

### Top Level

| Field      | Type     | Description                                                                         |
|------------|----------|-------------------------------------------------------------------------------------|
| diskguid   | GUID     | optional, the disk GUID. If not given, or full zeros, it will be generated          |
| disksize   | integer  | optional, the size of the disk image in Megabytes. If not given, it is calculated   |
| align      | integer  | optional, the partition alignment in Kilobytes. Zero gives sector alignment         |
| iso9660    | boolean  | optional, wether to generate ISO9660 Boot Catalog into the image. Defaults to false |
| config     | filename | BOOTBOOT configuration file. It is parsed for the kernel filename                   |
| initrd     | struct   | the initial ramdisk's definition, see below                                         |
| partitions | array    | partition definitions, see below                                                    |

Example:
```
{
    "diskguid": "00000000-0000-0000-0000-000000000000",
    "disksize": 128,
    "align": 1024,
    "iso9660": true,
    "config": "boot/sys/config",
    "initrd": { "type": "tar", "gzip": true, "directory": "boot" },
    "partitions": [
        { "type": "boot", "size": 16 },
        { "type": "ext4", "size": 128, "name": "Linux Exchange" },
        { "type": "ntfs", "size": 128, "name": "Windows Exchange" },
        { "type": "Microsoft basic data", "size": 32, "name": "MyOS usr", "file": "usrpart.bin" },
        { "type": "00000000-0000-0000-0000-000000000000", "size": 32, "name": "MyOS var", "file": "varpart.bin" }
    ]
}
```

### Initrd

| Field      | Type     | Description                                                                         |
|------------|----------|-------------------------------------------------------------------------------------|
| gzip       | boolean  | optional, wether to compress the initrd image, defaults to true                     |
| type       | string   | format of the initrd image. When invalid value given, it lists the options          |
| file       | filename | the filename of the image file to be used                                           |
| directory  | folder   | path to a folder, its contents will be used to generate the initrd                  |
| file       | array    | for multiarch images                                                                |
| directory  | array    | for multiarch images                                                                |

The fields `file` and `directory` are mutually exclusive. They can be both strings (if there's only one architecture),
or arrays (one array element for each architecture). Currently two architecture supported, which means there can be
two strings in the arrays. Which architecture is used depends on the kernel's architecture in that folder or image
file. Type is only mandatory for `directory`.

Examples:
```
    "initrd": { "file": "initrd.bin" },
    "initrd": { "type": "tar", "gzip": 0, "directory": "boot" },
    "initrd": { "gzip": true, "file": [ "initrd-x86.bin", "initrd-arm.bin" ] },
    "initrd": { "type": "cpio", "gzip": true, "directory": [ "boot/arm", "boot/x86" ] },
```

### Partitions

It is somewhat unusual, as the first array element is different than the rest. It specifies the boot partition,
therefore it has different types, and `file` / `directory` and `name` are not interpreted because that partition image is
always dinamically generated with the implicit name of "EFI System Partition". For the same reason, `size` is mandatory
for the first (boot) partition.

| Field      | Type     | Description                                                                         |
|------------|----------|-------------------------------------------------------------------------------------|
| size       | integer  | optional, the size of the partition in Megabytes. If not given, it is calculated    |
| file       | filename | optional, path to a partition image to be used                                      |
| directory  | folder   | optional, path to a folder, its contents will be used to generate the partition     |
| type       | string   | format of the partition. When invalid value given, it lists the options             |
| name       | string   | UTF-8 partition names, limited to UNICODE code points 32 to 65535 (BMP)             |

For the first entry, valid values for `type` are: `boot` (or explicit `fat16` and `fat32`). The utility handles these
comfortably, it tries to use FAT16 if possible to save storage space. There's a minimal size for the boot partition,
16 Megabytes. Although both the image creator and BOOTBOOT is capable of handling smaller sizes, some UEFI firmware
incorrectly assumes FAT12 when there are too few clusters on the file system. If the partition size is bigger than
128 Megabytes, then it automatically switches to FAT32. If you don't use `iso9660`, then you can also set FAT32 for
smaller images, but at least 33 Megabytes (that's a hard lower limit for FAT32). With `iso9660`, each cluster must
be 2048 bytes aligned, which is achieved by 4 sectors per cluster. The same problem applies here, both the image
creator and the BOOTBOOT loader capable of handling FAT32 with smaller cluster numbers, but some UEFI firmware don't,
and falsely assumes FAT16. To guarantee the minimum number of clusters, with ISO9660 and FAT32 the boot partition's
minimum size is 128 Megabytes (128*1024*1024/512/4 = 65536, just one larger than what fits in 16 bits).

For the other entries (starting from the second), `type` is either a GUID or one of a pre-defined file system aliases.
With an invalid string, the utility will list all possible values.

Example:
```
mkbootimg: partition #2 doesn't have a valid type. Accepted values:
  "65706154-4120-6372-6968-766520465320" / "tar"
  "5A2F534F-0000-5346-2F5A-000000000000" / "FS/Z"
  "6A898CC3-1DD2-11B2-99A6-080020736631" / "ZFS"
  "EBD0A0A2-B9E5-4433-87C0-68B6B72699C7" / "ntfs"
  "0FC63DAF-8483-4772-8E79-3D69D8477DE4" / "ext4"
  "516E7CB6-6ECF-11D6-8FF8-00022D09712B" / "ufs"
  "C91818F9-8025-47AF-89D2-F030D7000C2C" / "p9"
  "D3BFE2DE-3DAF-11DF-BA40-E3A556D89593" / "Intel Fast Flash"
  "21686148-6449-6E6F-744E-656564454649" / "BIOS boot"
     ...
  "77719A0C-A4A0-11E3-A47E-000C29745A24" / "VMware Virsto"
  "9198EFFC-31C0-11DB-8F78-000C2911D1B8" / "VMware Reserved"
  "824CC7A0-36A8-11E3-890A-952519AD3F61" / "OpenBSD data"
  "CEF5A9AD-73BC-4601-89F3-CDEEEEE321A1" / "QNX6 file system"
  "C91818F9-8025-47AF-89D2-F030D7000C2C" / "Plan 9 partition"
  "5B193300-FC78-40CD-8002-E86C45580B47" / "HiFive Unleashed FSBL"
  "2E54B353-1271-4842-806F-E436D6AF6985" / "HiFive Unleashed BBL"
  ...or any non-zero GUID in the form "%08X-%04X-%04X-%04X-%12X"
```

If `file` given, then the partition is filled with data from that file. If `size` is not given or smaller than
the file's size, then the file's size will be the partition's size. If both given, and `size` is larger, then the
difference is filled up with zeros. Partition sizes will always be multiple of `align` Kilobytes. Using 1024
as alignment gives you 1 Megabyte aligned partitions. For the first entry, only `size` is valid, `file` isn't.
Alternatively to `file`, you might also able to use `directory` to generate the partition image from the contents
of a directory. This option is only available if the file system driver is implemented for `type`. Because there might
be no one-to-one relation between file system types and partition types, you can use `typeguid` to explicily set the
latter. This is only relevant when the `directory` directive is used. For example:
```
    { "type": "FS/Z", "typeguid": "5A2F534F-8664-5346-2F5A-000075737200", "size": 32, "name": "MyOS usr", "directory": "myusr" },
```

Finally, `name` is just an UTF-8 string, name of the partition. Maximum length is 35 characters. Not valid for the first entry.

Adding More File Systems
------------------------

These are listed in the fs registry, in the file `fs.h`. You can freely add new types. For file systems that you
want to use for generating partition images or initrd as well, you must implement three functions, like:

```
void somefs_open(gpt_t *gpt_entry);
void somefs_add(struct stat *st, char *name, unsigned char *content, int size);
void somefs_close();
```

The first is called whenever a new file system is to be created. The `gpt_entry` is NULL when called for initrd creation.
As the given directory is recursively parsed, for each directory entry an "add" call is made. This should add the file or
directory to the file system image. Here `st` is the stat struct for the file, `name` is the filename with full path,
`content` and `size` are the file's content, or in case of a symbolic link, the pointed path. Finally when the parsing is
done, the close function is called to finalize the image. Only the "add" function is mandatory, the other two are optional.

These functions can use two global variables, `fs_base` and `fs_len` which holds the buffer for the filesystem image
in memory.

In lack of these functions, the file system still can be used in the partition's `type` field, but then only the GPT entry
will be created, not the content of the partition.

Keeping the built-in binaries up-to-date
----------------------------------------

To avoid dependencies, the image creator includes all the necessary binaries. If these are updated, then delete data.c
and run `make` which will regenerate it. If there are missing files, then in the `aarch64-rpi` directory run `make getfw`,
that will download the latest Raspberry Pi firmware files. Then `make` in this directory should run without problems.
