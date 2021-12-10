#Build a customized RPi3 bootable image with Docker from macOS Monterey (and run it in QEMU)

###Prerequisites

- macOS BigSur or Monterey, with brew tool installed (see [https://brew.sh/](https://brew.sh/) for more info)
- Docker Desktop 4.3.0 or later
- Xcode 13.1 or above
- macFUSE 4.2.3 or above from here: [https://github.com/osxfuse/osxfuse/releases](), once installed you will have to restart the machine.

####Packages to install from brew

1. Install *qemu* to emulate the docker RPi images:

	```
	$ brew install qemu
	```

2. Install *e2fsprogs* to manage ext4 volumes mounted:

	```
	$ brew install e2fsprogs
	```

#### Other packages necessaries

Build *fuse-ext2* to inspect linux images in macOS. To build that package you have to follow the official guide for macOS from [https://github.com/alperakcan/fuse-ext2](https://github.com/alperakcan/fuse-ext2). They provide an script to build the package, that script will download all the dependecies necesaries if they are not installed in the system.

### [OPTIONAL] - Run a raspbian-lite image in QEMU
You should check if your system is ready to run a Raspbian strecth image in QEMU. To check it follow the next steps:

1. Prepare a work folder and download the last image for Raspbian stretch, then unzip:

	```
	$ mkdir -p test
	$ cd test
	$ wget https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/2019-04-08-raspbian-stretch-lite.zip
	$ unzip 2019-04-08-raspbian-stretch-lite.zip
	```

2. Download the QEMU kernel for Raspbian jessie (valid for stretch also):

	```
	$ wget https://raw.githubusercontent.com/dhruvvyas90/qemu-rpi-kernel/master/kernel-qemu-4.4.34-jessie
	```

3. Launch the image in QEMU with the next command:

	```
	$ qemu-system-arm -kernel kernel-qemu-4.4.34-jessie -cpu arm1176 -m 256M -M versatilepb -serial stdio -append "root=/dev/sda2 rootfstype=ext4 rw" -hda 2019-04-08-raspbian-stretch-lite.img -no-reboot
	```

If everything works well QEMU should run the image in your local machine.

## Build the custom RPi bootable image

1. Prepare a work folder and download the last image for Raspbian stretch, then unzip:

	```
	$ mkdir -p docker-rpi
	$ cd docker-rpi
	$ wget https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/2019-04-08-raspbian-stretch-lite.zip
	$ unzip 2019-04-08-raspbian-stretch-lite.zip
	```

2. Make a copy for the unzipped image and rename it to *raspbian.img*. That copy will be our referenced image in the guide and later will be modified:

	```
	$ cp 2019-04-08-raspbian-stretch-lite.img raspbian.img
	```

3. With *hdiutil* inspect the image to see where the partitions will be mapped on the system. We are especially interested in the root partition (*/dev/disk4s2* in my case):

	```
	$ hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount raspbian.img
	/dev/disk4          	FDisk_partition_scheme
	/dev/disk4s1        	Windows_FAT_32
	/dev/disk4s2        	Linux
	```

4. Mount the Raspbian root Linux partition with *fuse-ext2*:

	```
	$ mkdir -p mnt/raspbian
	$ fuse-ext2 /dev/disk4s2 mnt/raspbian
	```

5. Create a tar file with the content of that mounted volume ( *--acls --xattrs* params will reserve extended file permissions):

	```
	$ sudo tar --acls --xattrs -cvf root.tar -C mnt/raspbian .
	```

6. Create a Dockerfile with the next content:

	```
	FROM scratch

	USER root
	ADD root.tar /

	RUN echo "pi:test" | chpasswd
	```
	With this Dockerfile we are modifying the image so that we change the password of the pi user. When we boot the image with QEMU we will verify that the user pi has the new password "test".

7. Generate the image with the next docker command:

	```
	$ docker build -t mytag/raspbian:latest --platform linux/arm/v7 --load .
	```

8. Now the modified file system is hanging out in the Docker image tagged raspi-custom. Next, this file system needs to be exported back out into a tar file: 

	```
	$ CID=$(docker run -d --platform linux/arm/v7 mytag/raspbian:latest /bin/true)
	$ docker export -o custom-root.tar ${CID}
	$ docker container rm ${CID}
	```

9. We need to overwrite the original root filesystem with the modified one. The next commands will reformat the root file system and copy the modified one into its place.

	```
	$ umount /dev/disk4s2
	$ sudo $(brew --prefix e2fsprogs)/sbin/mkfs.ext4 /dev/disk4s2
	$ fuse-ext2 /dev/disk4s2 mnt/raspbian -o rw+
	$ sudo tar xvf custom-root.tar -C mnt/raspbian
	$ umount mnt/raspbian
	```

10. Test the output image in QEMU:

	```
	$ wget https://raw.githubusercontent.com/dhruvvyas90/qemu-rpi-kernel/master/kernel-qemu-4.4.34-jessie
	$ qemu-system-arm -kernel kernel-qemu-4.4.34-jessie -cpu arm1176 -m 256M -M versatilepb -serial stdio -append "root=/dev/sda2 rootfstype=ext4 rw" -hda raspbian.img -no-reboot
	```
