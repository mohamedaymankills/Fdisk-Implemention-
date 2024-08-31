
# `fdisk_l` Implementation in C

## Overview

This project implements a simplified version of the `fdisk -l` command in C. The program reads the partition table from the Master Boot Record (MBR) of a disk and displays disk and partition information in a format similar to the `fdisk -l` command. It supports showing primary, extended, and logical partitions.

## Features

- Displays the disk size in GiB, total bytes, and sectors.
- Shows partition details such as start sector, end sector, total sectors, and size in GiB.
- Supports both primary and logical partitions (extended partitions).

## Requirements

- A Linux system with a C compiler (e.g., GCC).
- Permissions to read disk devices (typically requires `sudo`).

## Compilation

To compile the program, use the following command:

```sh
gcc -o fdisk_l fdisk_l.c
```

## Usage

Run the program with the path to a disk device as an argument. For example:

```sh
sudo ./fdisk_l /dev/sda
```

**Note:** You must run this program with sufficient privileges to access the raw disk device.

## Output Format

The program will produce an output similar to the following:

```
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Disklabel type: dos
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Device        Start       End   Sectors   Size
/dev/sda1      2048   1026047   1024000   0.5G
/dev/sda2   1026048  83886079  82860032  39.5G
```

## Code Explanation

### Header Files

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/ioctl.h>
#include <linux/fs.h>
#include <linux/hdreg.h>
```

These headers include standard I/O functions, data types, file operations, memory operations, and Linux-specific ioctl functions.

### Constants and Structures

```c
#define SECTOR_SIZE 512
#define PARTITION_TABLE_OFFSET 446
#define PARTITION_ENTRY_SIZE 16
#define PARTITION_TABLE_ENTRIES 4
#define MBR_SIGNATURE_OFFSET 510
#define MBR_SIGNATURE 0xAA55

#pragma pack(push, 1)

typedef struct {
    uint8_t status;
    uint8_t first_chs[3];
    uint8_t type;
    uint8_t last_chs[3];
    uint32_t first_lba;
    uint32_t sectors;
} PartitionEntry;

#pragma pack(pop)
```

- **Constants:** These define the sector size, MBR offsets, and signature.
- **PartitionEntry Structure:** This structure represents a partition entry in the MBR. The `#pragma pack` directive ensures no padding is added, maintaining compatibility with the MBR format.

### Reading the MBR

```c
void read_mbr(int fd, PartitionEntry *partitions) {
    uint8_t mbr[SECTOR_SIZE];

    if (lseek(fd, 0, SEEK_SET) < 0) {
        perror("lseek");
        exit(EXIT_FAILURE);
    }

    if (read(fd, mbr, SECTOR_SIZE) != SECTOR_SIZE) {
        perror("read");
        exit(EXIT_FAILURE);
    }

    if (*((uint16_t *)&mbr[MBR_SIGNATURE_OFFSET]) != MBR_SIGNATURE) {
        fprintf(stderr, "Invalid MBR signature.
");
        exit(EXIT_FAILURE);
    }

    memcpy(partitions, &mbr[PARTITION_TABLE_OFFSET], PARTITION_TABLE_ENTRIES * PARTITION_ENTRY_SIZE);
}
```

- **`lseek` and `read`:** These functions are used to move to the beginning of the disk and read the MBR.
- **Signature Check:** Verifies the MBR signature (0xAA55) to ensure the data read is valid.
- **Partition Table Extraction:** The partition table entries are extracted from the MBR.

### Printing Partition Information

```c
void print_partition(PartitionEntry *partition, int num, const char *device) {
    printf("%s%d %10u %10u %9u %6.1fG
", device, num,
           partition->first_lba,
           partition->first_lba + partition->sectors - 1,
           partition->sectors,
           (float)partition->sectors * SECTOR_SIZE / (1024 * 1024 * 1024));
}
```

- **Formatting:** This function prints the partition details, including start and end sectors, total sectors, and size in GiB.

### Reading Logical Partitions

```c
void read_logical_partitions(int fd, uint32_t ebr_lba, const char *device) {
    PartitionEntry partitions[2];
    int logical_num = 5;  // Assuming first logical partition starts from /dev/sda5

    while (ebr_lba) {
        if (lseek(fd, ebr_lba * SECTOR_SIZE, SEEK_SET) < 0) {
            perror("lseek");
            exit(EXIT_FAILURE);
        }

        if (read(fd, partitions, 2 * PARTITION_ENTRY_SIZE) != 2 * PARTITION_ENTRY_SIZE) {
            perror("read");
            exit(EXIT_FAILURE);
        }

        if (partitions[0].sectors != 0) {
            print_partition(&partitions[0], logical_num++, device);
        }

        ebr_lba = partitions[1].first_lba ? ebr_lba + partitions[1].first_lba : 0;
    }
}
```

- **Extended Boot Record (EBR):** This function reads the EBR to find logical partitions within an extended partition.
- **Loop:** Iterates through all logical partitions by following the chain of EBRs.

### Printing Disk Information

```c
void print_disk_info(int fd, const char *device) {
    struct hd_geometry geo;
    unsigned long long size;

    // Get disk size in bytes
    if (ioctl(fd, BLKGETSIZE64, &size) < 0) {
        perror("ioctl");
        exit(EXIT_FAILURE);
    }

    // Get disk geometry (heads, sectors, cylinders)
    if (ioctl(fd, HDIO_GETGEO, &geo) < 0) {
        perror("ioctl");
        exit(EXIT_FAILURE);
    }

    printf("Disk %s: %.1f GiB, %llu bytes, %llu sectors
",
           device, (float)size / (1024 * 1024 * 1024), size, size / SECTOR_SIZE);

    printf("Disklabel type: dos
");
    printf("Units: sectors of 1 * %d = %d bytes
", SECTOR_SIZE, SECTOR_SIZE);
    printf("Sector size (logical/physical): %d bytes / %d bytes
", SECTOR_SIZE, SECTOR_SIZE);
    printf("I/O size (minimum/optimal): %d bytes / %d bytes
", SECTOR_SIZE, SECTOR_SIZE);
    printf("
");
    printf("Device        Start       End   Sectors   Size
");
}
```

- **`ioctl` Calls:** These calls retrieve disk size and geometry information.
- **Formatted Output:** The disk information is printed in a human-readable format.

### Main Function

```c
int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <device>
", argv[0]);
        exit(EXIT_FAILURE);
    }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    PartitionEntry partitions[PARTITION_TABLE_ENTRIES];
    read_mbr(fd, partitions);

    print_disk_info(fd, argv[1]);

    for (int i = 0; i < PARTITION_TABLE_ENTRIES; i++) {
        if (partitions[i].sectors != 0) {
            print_partition(&partitions[i], i + 1, argv[1]);
        }

        if (partitions[i].type == 0x05 || partitions[i].type == 0x0F) {  // Extended partition
            read_logical_partitions(fd, partitions[i].first_lba, argv[1]);
        }
    }

    close(fd);
    return 0;
}
```

- **Command-Line Argument:** The program expects the disk device as a command-line argument.
- **MBR and Partitions:** The MBR is read, and both primary and logical partitions are printed.
- **File Descriptor:** The disk device is opened, and the file descriptor is passed to various functions.

## Potential Enhancements

- **Support for GPT:** Extend the program to support GPT (GUID Partition Table) in addition to MBR.
- **Error Handling:** Improve error handling for various edge cases (e.g., corrupted MBR).
- **Partition Types:** Add detailed descriptions for different partition types.
- **Filesystem Detection:** Implement filesystem detection for partitions.

## License

This project is licensed under the MIT License.
