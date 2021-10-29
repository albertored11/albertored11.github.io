---
title: SSD + HDD storage setup in a Linux desktop with bind mount
date: 2021-10-29 18:36:00 +0200
categories: [Linux, System administration]
tags: [linux, storage, hdd, sdd, fstab, bind, mount]
---

Hi everyone! Today I'm going to share with you how I have set up my storage in my **desktop PC** using a **250 GB SSD** and a **2 TB HDD**.

## Background

Back in august 2020, I built my own desktop PC with the help of a friend, and the storage I decided to use was:

* a Samsung MZ-V7S250BW 250 GB NVMe M.2 SSD
* and a Seagate ST2000DM008 Barracuda 2TB HDD

And my choice for the main OS was **Linux** (specifically, Arch Linux).

### First idea: EFI and root in SSD, home in HDD

Before building my PC, I had been thinking about the **partition layout** I was going to use. After doing some research on the internet, my initial idea was to use the **SSD** for the **EFI System Partition**, mounted in `/boot`, and the **root partition**, mounted in `/`, and the **HDD** for **a single partition** mounted in `/home`.

The idea was to store the **system files**, which usually need a fast access, in the **fastest drive** (and also the smaller and the most expensive), the **SSD**, and the **user files**, which are mostly less frequently accessed and which access speed is not critical, in the **slowest drive** (and also the biggest and the cheapest), the **HDD**.

The **filesystem** I decided to use for the `/` and `/home` partitions was **ext4**, since it is **simple and rock-solid**. I'm curious about other filesystems like btrfs, which have really interesting features, but at the moment I'm sticking to ext4.

So, when my friend and I finished building the PC, I did a first Arch Linux install with the layout I just described. However, when I booted up my machine after installing the graphical enviroment, I noticed **the configuration didn't load correctly**. I assumed that was because the **configuration files** need a **fast access** just like the system files, and the **user config files** are stored in `$HOME/.config`, which was a subdirectory of the mountpoint of the **HDD** (`/home`).

### Second idea: EFI and root in SSD, HDD single partition mounted in home and symlink dirs

Then, I did a **second installation** from scratch, using a **different partition layout**: just using the SSD like the previous time and **no separate partition** for `/home`. I would figure out later what to do with the HDD.

The idea I came up with was **mounting the HDD in a subdirectory** under my `$HOME` directory called `.hdd`, and inside making directories for **every standard XDG user directory I wanted to be stored in the HDD** (`Documents`, `Music`, `Pictures` and `Videos`). The rest of the directories (`Downloads`, hidden directories such as `.config`, `.cache`, etc.) would be stored in the **SSD**. Then, I would **symlink** those directories in `.hdd` to `$HOME/Documents`, `$HOME/Music`, etc.

At first it seemed like it was a good setup and worked as intended, but later I realized **it wasn't**. Why?

* Mounting a device that could be used at a system level in a hidden user directory sounds **dirty**.
* Using the symlinked directories was a **problem** for certain things.

### Final idea: EFI and root in SSD, HDD single partition mounted in root and bind mount dirs

#### Bind mount

I wasn't satisfied with my previous setup, so I did a bit more research and heard about **bind mount**, which is basically **mounting a subdirectory** from a partition that is **already mounted** in **another directory**. This, unlike symlinks, doesn't work at a **filesystem** level, so we're not actually changing anything in our filesystems. Instead, we are showing **the same directory** in two **different points**.

This means that we can mount a partition somewhere in our root filesystem and then **selectively** remount directories in it in another part of our **root filesystem** or even in **another filesystem** so they are available in **both locations**.

For example, let's say we have a directory under `/home/myuser/Pictures/MyDownloadedPics/Wallpapers` that we want to be available at `/home/otheruser/MyWallpapers` as well. We can **bind mount** the first directory in the latter... and that's it! We can access the **same directory** using two **different locations**, and **any modification** on one directory is **immediately effective** on the other one, as they are showing the **same data**.

#### Setup

So the final idea for my setup was **mounting the HDD** in `/mnt/home`, creating a directory for my user, and inside it, the **XDG user directories** I wanted in the **HDD** (`Documents`, `Music`, `Pictures` and `Videos`). Finally, I would **bind mount** those directories in `$HOME/Documents`, `$HOME/Music`, etc.

That way, I would have everything stored in the SDD but the **user directories** which are usually used to store **large files** (or a large number of files) that **doesn't require fast access** (pictures, videos, music, etc.).

I started using this setup several months ago and I think it is just **the best solution**. Of course, this is totally **subjective** and there are probably other and better ways to do this, so feel free to **let me know** if that is the case.

## Tutorial

So... how can we achieve **this setup**?

I'm going to explain it **step by step**, covering both the cases of a **new** and an **existing** Linux-based installation.

I'm assuming the **device file** for the SSD is `/dev/nvme0n1` and the device file for the HDD is `/dev/sda`, which is my case and **may** be yours, but it also **may not be**, so replace them if necessary.

### New installation

#### Create and format partitions

If we are in the case of a **new installation**, the first thing we have to do when it is required during the installation process is to **create the partitions and the filesystems**.

We are using the `cfdisk` tool with this purpose. If your distro uses a graphical installer, create the described partition layout using the appropriate tool.

First, we're going to proceed with the **SSD**:

```bash
cfdisk /dev/nvme0n1
```

Create a **GPT** partition table.

Create a new partition for the **EFI System Partition**. A size of `512M` should be more than enough. Set the partition type to **EFI System**.

Create a new partition for the **root filesystem**. Use the remaining size.

Write changes and quit.

Format partitions accordingly (**FAT32** for EFI System Partition as required, and **ext4** for the root partition for simplicity):

```bash
mkfs.ext4 /dev/nvme0n1p2
mkfs.fat -F32 /dev/nvme0n1p1
```

Then, with the **HDD**:

```bash
cfdisk /dev/sda
```

Create a **GPT** partition table.

Create a new partition for `/mnt/home`. Use the total size of the disk.

Write changes and quit.

Format the newly created partition:

```bash
mkfs.ext4 /dev/sda1
```

#### Mount partitions

First, **mount** the root partition:

```bash
mount /dev/nvme0n1p2 /mnt
```

Create the **mountpoints** for the other partitions:

```bash
mkdir -p /mnt/boot /mnt/mnt/home
```

and **mount** them:

```bash
mount /dev/nvme0n1p1 /mnt/boot
mount /dev/sda1 /mnt/mnt/home
```

#### Prepare bind mountpoints

When the installation process is finished and we have successfully booted up our system and created and logged in as an **unprivileged user**, we can move on to the exciting part: setting up **bind mount**!

First, let's create the standard **XDG user directories** (you may will the `xdg-user-dirs` package):

```bash
xdg-user-dirs-update
```

Then, create the directories we're going to **bind mount** to our `$HOME` in our **HDD**. My **personal choice** is `Documents`, `Music`, `Pictures` and `Videos`, but feel free to make yours **as needed**. Think of which directories are you going to store **large files** (or a large number of files) that **don't need fast access** in.

First, **create** a directory for our user in `/mnt/home` and **change** the ownership so the directory belongs to it:

```bash
cd /mnt/home
sudo mkdir -p myuser
sudo chown -R myuser:myuser myuser
```

and after that **create** the directories themselves:

```bash
cd myuser
mkdir -p Documents Music Pictures Videos
```

#### Define bind mountpoints

Finally, **edit** the `/etc/fstab` file, which describes how **filesystems** have to be **mounted**. Add the following lines at the end (modify them as needed if you're using different directories):

```
# /mnt/home/myuser/Documents
/mnt/home/myuser/Documents  /home/myuser/Documents  none    rw,bind 0 0

# /mnt/home/myuser/Music
/mnt/home/myuser/Music      /home/myuser/Music      none    rw,bind 0 0

# /mnt/home/myuser/Pictures
/mnt/home/myuser/Pictures   /home/myuser/Pictures   none    rw,bind 0 0

# /mnt/home/myuser/Videos
/mnt/home/myuser/Videos     /home/myuser/Videos     none    rw,bind 0 0
```

Those lines are telling our system to **bind mount** `/mnt/home/myuser/Documents` to `/home/myuser/Documents`, `/mnt/home/myuser/Music` to `/home/myuser/Music` and so on.

Test the setup:

```bash
sudo mount -a
```

and try creating a **file** in one of the bind mounted directories. For example, if we create a file called `test` in `/home/myuser/Documents`, it **should appear** in `/mnt/home/myuser/Documents` as well.

If it works fine, **we're done**! On the next reboots, the bind mount will happen **automatically**.

And that's it! We will have everything stored in our **SSD**, excepting the files saved to our `Documents`, `Music`, `Pictures` and `Videos` user directories.

### Existing installation

But what if we want to use this setup in an **existing** Linux installation? The steps to follow depend on the situation: if you are already using your drive for your **home partition**, it will be a little more tricky.

#### Extra drive unused or not mounted in the home directory

Let's start with the easiest case: you just installed the **new drive** or you just were not using it, or it was mounted in your filesystem, but **not** in `/home`.

I'm assuming the drive is **already partitioned** and formatted with a single **ext4** partition which device file is `/dev/sda1`.

First of all, if the partition is mounted, **unmount** it:

```bash
sudo umount /dev/sda1
```

**Create** the mountpoint for the partition:

```bash
sudo mkdir -p /mnt/home
```

and **append** a couple of lines to `/etc/fstab` file to define the mount:

```bash
sudo bash -c 'echo "# /dev/sda1" >> /etc/fstab'
sudo bash -c 'echo -e "UUID=$(blkid -s UUID -o value /dev/sda1)\t/mnt/home\text4\t\trw,relatime\t0 2" >> /etc/fstab'
sudo bash -c 'echo >> /etc/fstab'
```

If the partition used to be mounted in a **different path**, remove those lines, too.

Check `/etc/fstab` file manually to verify everything is **correct** and **well-formatted** and **test** the configuration:

```bash
sudo mount -a
```

If it works properly, now follow the guide for a new installation starting from [here](#Prepare bind mountpoints) but, instead of creating directories in `/mnt/home/myuser`, move them from `/home/myuser` and create empty directories in the latter:

skip

```bash
cd myuser
mkdir -p Documents Music Pictures Videos
```

and instead do

```bash
mv /home/myuser/{Documents,Music,Pictures,Videos} /mnt/home/myuser
mkdir -p /home/myuser/{Documents,Music,Pictures,Videos}
```

#### Extra drive used for home partition

Another case would be that you are already using that **extra drive** and having it mounted in `/home`. How can we do the **migration** then?

First of all, **log out** as your user and **log in** as root in a **tty** (if you are using a desktop manager, hit Alt + F2 or Ctrl + Alt + F2 to switch to tty2). We're doing this because regular users have their home directory under `/home`, which we are about to **remount** in a different path, while root user has its home under `/root`, so that won't cause any problem.

I'm assuming the corresponding device for the former home partition is `/dev/sda1`.

**Unmount** the home partition:

```bash
umount /dev/sda1
```

**Create** the mountpoint for the partition:

```bash
mkdir -p /mnt/home
```

and edit `/etc/fstab` accordingly, replacing `/home` with `/mnt/home`, which is the new **mountpoint** for our partition:

```bash
sed -i 's/home/mnt\/home/g' /etc/fstab
```

Check `/etc/fstab` file manually to verify everything is **correct** and test the configuration:

```bash
mount -a
```

If it works properly, now **recreate** the home directory for your user in `/home` (and for every other regular user in your system):

```bash
mkdir -p /home/myuser
chown myuser:myuser /home/myuser
```

Then, we are going to **move** everything there except the directories you want to keep in your **extra drive**. First, enable a couple of **shell options** for glob matching:

```bash
shopt -s extglob
shopt -s dotglob
```

Those will let us match everything except certain files or directories, and match **hidden files** too when using `*` wildcard.

Then **move** everything inside our home directory except, for example, `Documents`, `Music`, `Pictures` and `Videos`, if those are the directories we want to keep in the **extra drive**:

```bash
mv /mnt/home/myuser/!(Documents|Music|Pictures|Videos) /home/myuser
```

Make **empty directories** under our home directory for those that are going to stay in the **extra drive** so we can **bind mount** them and make them belong to **your user**:

```bash
mkdir -p /home/myuser/{Documents,Music,Pictures,Videos}
chown myuser:myuser /home/myuser/{Documents,Music,Pictures,Videos}
```

Now, **reboot** your system and log in as your regular user and follow the guide for a new installation starting from [here](#Define bind mountpoints).

# Conclusion

And that's all! I hope you find this setup useful and, as always, reach me out at albertomost@gmail.com if you have any questions, suggestions, etc.
