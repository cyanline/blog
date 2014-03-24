## Fusion Drive and Core Storage Forensics
date: March 20 2014

Apple's new iMac ships with a storage medium named Fusion Drive. Fusion Drive is the marketing name for Apple's custom RAID solution called CoreStorage.

### 1. Acquiring a CoreStorage Volume

CoreStorage volumes pose some unique challenges to a forensic investigator.

#### Can you acquire a CoreStorage volume in a forensically sound manner?
It is possible to acquire a CoreStorage volume in a forensically sound manner.  There are a couple ways that we found to do this. Both methods require the source machine to be in target disk mode.

##### Method 1
The first method requires an imaging system that will not write to the drives, such as the [FDAS](http://www.cyanline.com/fdas.php), or software such as [MiniDAS](http://www.cyanline.com/minidas.php). Attach the imaging system to the machine in target disk mode and acquire each drive separately.  This method ensures that the original source drives are not changed in any way. Using an imaging system to acquire the drives separately provides a physical acquisition which will give the forensic examiner access to all active and deleted data. See below for further details.

##### Method 2
The second method requires another machine running Mac OS X.  Once the source machine is in target disk mode DO NOT attach the source machine to the second machine running Mac OS X, the second machine will mount the CoreStorage volume and potentially change it, making the acquisition not forensically sound. Attach the destination drive where the evidence will be stored.  The operating system will automatically mount the drive.  Once the drive is mounted run the command below and take note of which drives are presented, our destination drive is /dev/disk1.

    $ ls -l /dev/disk* | grep disk.$
    brw-r-----  1 root  operator    1,   0 Mar 18 16:34 /dev/disk0
    brw-r-----  1 root  operator    1,   4 Mar 19 15:37 /dev/disk1

Next you must run the command below to disable the system from mounting or ejecting any more disks.

    $ sudo launchctl unload /System/Library/LaunchDaemons/com.apple.diskarbitrationd.plist

You can now safely attach your evidence machine in target disk mode. Since we have disabled disk arbitration on the system we can no longer use diskutil to inspect which drives are attached. Running the following command should now show 3 additional disks if the CoreStorage volume is made up of 2 disks.

    $ ls -l /dev/disk* | grep disk.$
    brw-r-----  1 root  operator    1,   0 Mar 18 16:34 /dev/disk0
    brw-r-----  1 root  operator    1,   4 Mar 19 15:37 /dev/disk1
    brw-r-----  1 root  operator    1,   7 Mar 19 15:55 /dev/disk2
    brw-r-----  1 root  operator    1,  10 Mar 19 15:55 /dev/disk3
    brw-r-----  1 root  operator    1,  15 Mar 19 16:00 /dev/disk4

In our case above, /dev/disk2 and /dev/disk3 are the physical disks that make up the CoreStorage volume.  /dev/disk4 is the CoreStorage volume.

Now that we know which disk is the CoreStorage volume we can acquire the drive using a tool such as GNU's dd.

    $ dd if=/dev/disk4 of=/path/to/save/the/image/file.dd bs=4096

We have chosen a blocksize of 4096 as a result of our research into [forensic acquistion performance](http://highspeedforensics.com/).

#### Is a machine running Mac OS X required to assemble a CoreStorage volume?
As of right now there are no solutions available outside of the tools that come with Mac OS X to assemble a CoreStorage volume.

#### Is it possible to offer a CoreStorage volume as a block device on Linux?
It is possible to install Mac OS X on a virtual machine, if you use the virtual machine to assemble the CoreStorage volume you should be able to export the volume to the host operating system. We have not tried this yet.

#### Can we image each drive separately and assemble the images at a later time for examination?
Through our testing we have discovered that you can acquire the drives separately and reassemble them at a later date, doing so provides a more complete acquisition because it provides a physical read from the disk which returns both active and deleted data on the disk as well as the disk's controller information (e.g. hours on, power cycle count, etc). Once you have acquired each drive separately you can reassmble the CoreStorage volume using a machine running OS X.  To do so you must attach both images as drives to the machine using the following command.

    $ hdiutil attach -readonly -nomount -imagekey diskimage-class=CRawDiskImage image1.dd
    $ hdiutil attach -readonly -nomount -imagekey diskimage-class=CRawDiskImage image2.dd

Mac OS X will automatically assemble the CoreStorage volume but will not mount it.  From here you can image the CoreStorage volume.


### 2. Creating a CoreStorage Volume

A CoreStorage volume can be made up of 1 or more partitions on a block device or 1 or more block devices. In our testing we tried three different configurations.

1. Two 8 GB USB Drives
2. Two 8 GB USB Drives and a 750 GB SATA drive connected through a SATA to Thunderbolt adapter
3. One 8 GB USB Drive and a 750 GB SATA drive connected through a SATA to Thunderbolt adapter

To create a CoreStorage volume you must identify the drives that you would like to use to create the volume. To list the drives and their partitions run the following command:

    $ diskutil list
    /dev/disk0
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:      GUID_partition_scheme                        *1.0 TB     disk0
       1:                        EFI EFI                     209.7 MB   disk0s1
       2:                  Apple_HFS Macintosh HD            999.7 GB   disk0s2
       3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3
    /dev/disk1
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:      GUID_partition_scheme                        *8.0 GB     disk1
       1:                        EFI EFI                     209.7 MB   disk1s1
       2:          Apple_CoreStorage                         7.7 GB     disk1s2
       3:                 Apple_Boot Boot OS X               134.2 MB   disk1s3
    /dev/disk2
       3:                 Apple_Boot Boot OS X               134.2 MB   disk2s3

We will be using /dev/disk1 and /dev/disk2 for our CoreStorage volume.  Ensure that any data on these drives is backed up, the process of creating a CoreStorage volume marks all data on the drives as deleted. To create the volume run the command below. Be sure to replace the logical volume group name with your own name (any name works) and replace the disks specified with the disks you identified in the previous step.

    $ diskutil coreStorage create CyanLogicalVolumeGroup /dev/disk1 /dev/disk2
    Started CoreStorage operation
    Unmounting disk2
    Repartitioning disk2
    Unmounting disk
    Creating the partition map
    Rediscovering disk2
    Adding disk2s2 to Logical Volume Group
    Creating Core Storage Logical Volume Group
    Switching disk2s2 to Core Storage
    Waiting for Logical Volume Group to appear
    Discovered new Logical Volume Group "1766287B-3C0B-4B08-A5AD-EFA476D6E8ED"
    Core Storage LVG UUID: 1766287B-3C0B-4B08-A5AD-EFA476D6E8ED
    Finished CoreStorage operation

Take note of the CoreStorage Logical Volume Group UUID that is returned from the command above.  You will need it to create the final CoreStorage volume.

To create the final CoreStorage volume run the command below. Be sure to replace the CoreStorage Logical Volume UUID with the one that was generated by your output in the previous command. It appears that the only filesystem currently supported is Journaled HFS+ which is specified with the jhfs+ option.

    $ diskutil coreStorage createVolume 1766287B-3C0B-4B08-A5AD-EFA476D6E8ED jhfs+ CyanStorage 100%
    The Core Storage Logical Volume Group UUID is 1766287B-3C0B-4B08-A5AD-EFA476D6E8ED
    Started CoreStorage operation
    Waiting for Logical Volume to appear
    Formatting file system for Logical Volume
    Initialized /dev/rdisk3 as a 7 GB case-insensitive HFS Plus volume with a 8192k journal
    Mounting disk
    Core Storage LV UUID: 137CD0EC-A192-4F77-A13E-9ECA198BDE80
    Core Storage disk: disk3
    Finished CoreStorage operation

The operating system should automatically mount your newly created CoreStorage volume.  If you run the following command, then you will notice that a new block device is now present, which is the CoreStorage volume you have just created, in our case it is /dev/disk3.

    $ diskutil list
    /dev/disk0
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:      GUID_partition_scheme                        *1.0 TB     disk0
       1:                        EFI EFI                     209.7 MB   disk0s1
       2:                  Apple_HFS Macintosh HD            999.7 GB   disk0s2
       3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3
    /dev/disk1
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:      GUID_partition_scheme                        *8.0 GB     disk1
       1:                        EFI EFI                     209.7 MB   disk1s1
       2:          Apple_CoreStorage                         7.7 GB     disk1s2
       3:                 Apple_Boot Boot OS X               134.2 MB   disk1s3
    /dev/disk2
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:      GUID_partition_scheme                        *8.1 GB     disk2
       1:                        EFI EFI                     209.7 MB   disk2s1
       2:          Apple_CoreStorage                         7.8 GB     disk2s2
       3:                 Apple_Boot Boot OS X               134.2 MB   disk2s3
    /dev/disk3
       #:                       TYPE NAME                    SIZE       IDENTIFIER
       0:                  Apple_HFS CyanStorage            *7.4 GB     disk3

CoreStorage volumes are portable from one OS X Mavericks system to another.  For example if we were to attach both drives used to create this CoreStorage volume to another machine running Mac OS X the new machine would automatically detect the drives and mount the CoreStorage volume. This is done with the corestoragd daemon.

### 3. Reverse Engineering CoreStorage

Imaging the drives separately with an imaging system or creating a logical image of the CoreStorage volume with a Mac is acceptable but not ideal.  An ideal solution would be to physically image the entire volume at once.  To do this we must reverse engineer CoreStorage to provide us with physical access to the entire volume.


Once a CoreStorage volume is created we noticed at 0x0003030 on the second partition of each of the drives that make up the core storage volume, there exists information pertaining to the core storage volume

    $ sudo xxd /dev/disk2s2
    .
    .
    .
    0003000: 0000 0000 0000 0000 0018 0000 0000 0000  ................
    0003010: 0000 0000 0000 0000 0200 0000 0000 0000  ................
    0003020: 0108 0000 0000 0100 0120 0000 0000 0100  ......... ......
    0003030: 3c64 6963 743e 3c6b 6579 3e73 7364 2d75  <dict><key>ssd-u
    0003040: 6e69 742d 6e62 7974 6573 3c2f 6b65 793e  nit-nbytes</key>
    0003050: 3c69 6e74 6567 6572 2049 443d 2230 2220  <integer ID="0"
    0003060: 7369 7a65 3d22 3634 223e 3078 3230 3030  size="64">0x2000
    0003070: 303c 2f69 6e74 6567 6572 3e3c 6b65 793e  0</integer><key>
    0003080: 636f 6d2e 6170 706c 652e 636f 7265 7374  com.apple.corest
    0003090: 6f72 6167 652e 6c61 6265 6c2e 7365 7175  orage.label.sequ
    00030a0: 656e 6365 3c2f 6b65 793e 3c69 6e74 6567  ence</key><integ
    00030b0: 6572 2073 697a 653d 2233 3222 3e30 7831  er size="32">0x1
    00030c0: 3c2f 696e 7465 6765 723e 3c6b 6579 3e63  </integer><key>c
    00030d0: 6f6d 2e61 7070 6c65 2e63 6f72 6573 746f  om.apple.coresto
    00030e0: 7261 6765 2e6c 7667 2e75 7569 643c 2f6b  rage.lvg.uuid</k
    00030f0: 6579 3e3c 7374 7269 6e67 3e42 3636 3743  ey><string>B667C
    0003100: 4134 312d 4630 3132 2d34 4343 462d 3932  A41-F012-4CCF-92
    0003110: 4246 2d45 3146 4543 4237 3330 3839 443c  BF-E1FECB73089D<
    0003120: 2f73 7472 696e 673e 3c6b 6579 3e63 6f6d  /string><key>com
    0003130: 2e61 7070 6c65 2e63 6f72 6573 746f 7261  .apple.corestora
    0003140: 6765 2e6c 7667 2e6e 616d 653c 2f6b 6579  ge.lvg.name</key
    0003150: 3e3c 7374 7269 6e67 3e43 7961 6e46 7573  ><string>CyanFus
    0003160: 696f 6e3c 2f73 7472 696e 673e 3c6b 6579  ion</string><key
    0003170: 3e63 6f6d 2e61 7070 6c65 2e63 6f72 6573  >com.apple.cores
    0003180: 746f 7261 6765 2e6c 7667 2e73 7364 2d75  torage.lvg.ssd-u
    0003190: 6e69 742d 6e62 7974 6573 3c2f 6b65 793e  nit-nbytes</key>
    00031a0: 3c69 6e74 6567 6572 2049 4452 4546 3d22  <integer IDREF="
    00031b0: 3022 2f3e 3c6b 6579 3e63 6f6d 2e61 7070  0"/><key>com.app
    00031c0: 6c65 2e63 6f72 6573 746f 7261 6765 2e6c  le.corestorage.l
    00031d0: 7667 2e70 6879 7369 6361 6c56 6f6c 756d  vg.physicalVolum
    00031e0: 6573 3c2f 6b65 793e 3c61 7272 6179 3e3c  es</key><array><
    00031f0: 7374 7269 6e67 3e36 4532 4644 4332 372d  string>6E2FDC27-
    0003200: 4133 4136 2d34 3844 412d 4136 3631 2d41  A3A6-48DA-A661-A
    0003210: 3933 4334 3545 4242 3442 433c 2f73 7472  93C45EBB4BC</str
    0003220: 696e 673e 3c73 7472 696e 673e 3539 4530  ing><string>59E0
    0003230: 3242 3535 2d46 3943 372d 3446 4342 2d39  2B55-F9C7-4FCB-9
    0003240: 3638 352d 3142 3238 4641 3343 4445 4638  685-1B28FA3CDEF8
    0003250: 3c2f 7374 7269 6e67 3e3c 2f61 7272 6179  </string></array
    0003260: 3e3c 2f64 6963 743e 0000 0000 0000 0000  ></dict>........
    0003270: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    0003280: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    0003290: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    .
    .
    .

The UUID values in the hex dump match the UUID values returned by the following command:

    $ diskutil coreStorage list
    CoreStorage logical volume groups (1 found)
    |
    +-- Logical Volume Group B667CA41-F012-4CCF-92BF-E1FECB73089D
        =========================================================
        Name:         CyanFusion
        Status:       Online
        Size:         15415558144 B (15.4 GB)
        Free Space:   40960 B (41.0 KB)
        |
        +-< Physical Volume 6E2FDC27-A3A6-48DA-A661-A93C45EBB4BC
        |   ----------------------------------------------------
        |   Index:    0
        |   Disk:     disk2s2
        |   Status:   Online
        |   Size:     7660331008 B (7.7 GB)
        |
        +-< Physical Volume 59E02B55-F9C7-4FCB-9685-1B28FA3CDEF8
        |   ----------------------------------------------------
        |   Index:    1
        |   Disk:     disk3s2
        |   Status:   Online
        |   Size:     7755227136 B (7.8 GB)
        |
        +-> Logical Volume Family 55B965B1-5265-42B6-9D0A-42432C59D448
            ----------------------------------------------------------
            Encryption Status:       Unlocked
            Encryption Type:         None
            Conversion Status:       NoConversion
            Conversion Direction:    -none-
            Has Encrypted Extents:   No
            Fully Secure:            No
            Passphrase Required:     No
            |
            +-> Logical Volume 352C52AB-6A80-46F4-BE52-EDBABEBF93BB
                ---------------------------------------------------
                Disk:                  disk4
                Status:                Online
                Size (Total):          10231087104 B (10.2 GB)
                Conversion Progress:   -none-
                Revertible:            No
                LV Name:               CyanFusion
                Volume Name:           CyanFusion
                Content Hint:          Apple_HFS

This tells us that each drive that makes up a CoreStorage volum knows its own UUID as well as the UUID of all the other drives in the CoreStorage volume.  We believe the corestoraged daemon looks for these values and does not assemble a CoreStorage volume until all of the drives with UUIDs present in this configuration are attached to the machine.  This allows for the CoreStorage volumes to be moved from one machine to the other without the use of configuration files on the host machine.

In this writeup we have covered some of the basics of Apple's Fusion Drive/Core Storage, current solutions for forensically acquiring images of the Fusion Drive, and finally, started to collect information in the aim of mounting a Fusion Drive GNU/Linux side. Using this information and continuing research we hope to offer a complete solution soon.
