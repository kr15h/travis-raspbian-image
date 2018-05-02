# travis-raspbian-image

[![Build Status](https://travis-ci.org/kr15h/travis-raspbian-image.svg?branch=master)](https://travis-ci.org/kr15h/travis-raspbian-image)

Test repo for creating customised Raspberry Pi images with Travis CI

Using [this guide](https://disconnected.systems/blog/custom-rpi-image-with-github-travis/) as the main reference.

Read [Travis CI docs on GitHub releases](https://docs.travis-ci.com/user/deployment/releases/) for more details regarding upload.

## Mounting the Image

In order to modify the Raspbian disk image, we want to mount it as a loop device. For that we will use the `losetup` command, once can read a detailed manual about it [in the losetup man pages](https://linux.die.net/man/8/losetup).

In order to use `losetup` I need to get the byte offset of the image file. It is where the file system starts. One can use `fdisk -l disk.img` to get all the information (the -l is L, not one).

```
fdisk -l disk.img
``` 

One should get a similar output as the one following.

```
Disk Stick.img: 3984 MB, 3984588800 bytes
249 heads, 6 sectors/track, 5209 cylinders, total 7782400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0004bfaa

    Device Boot      Start         End      Blocks   Id  System
Stick.img1   *         128     8015999     4007936    b  W95 FAT32
```

It outputs essential information about the file system stored in the .img file and a table of partitions, **Start** and **End** sectors for each. Sectors are blocks of bytes. In order to get the **byte offset**, we need to know the **Sector size**. It is easy to spot that in the output, but how do we automate that?

This where `grep` and `sed` comes handy. With `grep` you can find the line that contains a specific text. Like so.

```
fdisk -l disk.img | grep "Sector size"
```

You can read more about grep on the [grep man page](https://www.gnu.org/software/grep/manual/grep.html). Here we also use the concept of the **Unix pipe** which is this symbol **|**. Anything that the `fdisk -l disk.img` command outputs is being forwarded to `grep "Sector size"` as an input. You should get a line similar to the one below.


```
Sector size (logical/physical): 512 bytes / 512 bytes
```

How do we get the **512** part? We can use `sed`. The following command removes anything that is not a number and returns the output.

```
sed -e s/[^0-9]//g
```

However this will return 512 two times.

```
512512
```

This is where I searched for and found a [sed tutorial](http://www.grymoire.com/Unix/Sed.html)! `sed` uses regular expressions to search for a pattern. One can use [regexr](https://regexr.com/) to make it easier to create regular expressions.

Basic `sed` syntax.

```
sed -r "s/search/replace"
```

The `-r` (`-E` on OS X) part enables serious regular expressions. By default `sed` has limitations. In order to get the **Sector size** I used the following `sed` command.

```
sed -r "s/[^0-9]*([0-9]+).*/\1/"
```

Here we are looking for a pattern that starts with anything that is not a number (`[^0-9]*`) a number that consists of one or more digits (`[0-9]+`) followed by zero or more characters of any kind (`.*`).

The full version of getting the **Sector size** from output of `fdisk -l disk.img` looks like the following.

```
fdisk -l opm.img | grep "Sector size" | sed -r "s/[^0-9]*([0-9]+).*/\1/"
```

It outputs `512`. Great! We can store that in a variable. To do that, we need to use the `$(...)` syntax to enclose the expression between round brackets.

```
secsize=$(fdisk -l opm.img | grep "Sector size" | sed -r "s/[^0-9]*([0-9]+).*/\1/")
```

Now we can use `echo $secsize` to print out the value.

Next we need to get the **Start** sectors for two of the partitions in the image file. In a regular Raspbian image there are two partions, one is used during boot (the `/boot` partition) and the other one is where the files are.

We will use the following `grep` command to get information about the first partition.

```
fdisk -l opm.img | grep ".img1"
```

And the following, to get information about the second partition.

```
fdisk -l opm.img | grep ".img2"
```

Next, we need to get the first number preceding a space character. We are going to use `sed` for this once more. This is a bit more complicated since the line we get from grep begins with text that has numbers after it. We are looking for the first number after that text. 

To define the first part we use `[^0-9]*[0-9][^0-9]` which means anything that is not a number (`[^0-9]*`) followed by a single digit (`[0-9]`) and then again with anything that is not a number (`[^0-9]`). The rest of it is the same as for getting the **Sector size**.

```
sed -r "s/[^0-9]*[0-9][^0-9]([0-9]+).*/\1/"
```

The following two lines we can now use to get **Start** sectors for both partitions.

```
start1=$(fdisk -l opm.img | grep ".img1" | sed -r "s/[^0-9]*[0-9][^0-9]([0-9]+).*/\1/")
start1=$(fdisk -l opm.img | grep ".img2" | sed -r "s/[^0-9]*[0-9][^0-9]([0-9]+).*/\1/")
```

Now we use multiplication to get the offset bytes for both partitions. We have to use the bash **arithmetic expansion operator**.

```
off1="$(($secsize*$start1))"
off2="$(($secsize*$start2))"
```

And lastly we can mount or .img file!!!

```
mkdir mnt
mount -o loop,offset=$off2 opm.img mnt
mount -o loop,offset=$off1 opm.img mnt/boot
```

[Here is another guide](https://wiki.debian.org/RaspberryPi/qemu-user-static) that includes how you can modify the disk image size (very important, since I want to install a lot of packages!!!)

First you would need to add some bytes to the disk image. In this example we add 1G.

```
dd if=/dev/zero bs=1M count=1024 >> opm.img
```

Then we want to moun


