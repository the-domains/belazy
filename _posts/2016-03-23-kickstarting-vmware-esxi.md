---
inFeed: true
hasPage: true
inNav: false
inLanguage: null
starred: false
keywords: []
description: Kickstarting VMware ESXi
datePublished: '2016-03-23T17:13:29.272Z'
dateModified: '2016-03-23T01:26:14.795Z'
title: ''
author: []
authors: []
publisher:
  name: null
  domain: null
  url: null
  favicon: null
sourcePath: _posts/2016-03-23-kickstarting-vmware-esxi.md
published: true
url: kickstarting-vmware-esxi/index.html
_type: Article

---
Kickstarting VMware ESXi

https://michael.lustfield.net/misc/completely-automated-esxi-deployment

This post has to come with the disclaimer that 99.97% of ESXi deployments have_NO_use for anything this complicated. For that 0.03% of us, ESXi can be one heck of a deployment nightmare. It doesn't have to be, though.

## Background: The Situation

The company I work for has hundreds of remote facilities and each of these facilities connects back to our main office. Most of the time this is a pair of T1 lines, but sometimes it's just a single line. Each facility needs local file storage. They need other things locally, but the file storage is easy to put into perspective.

The easy answer is to deploy a server at each facility that keeps data locally and synchronizes with our main office.

Well, we will want to replace this hardware someday so obviously we'll want virtualization. There are many options out there and we almost went with proxmox but in the end settled on ESXi.

So now we will have an ESXi host at each facility and virtual machines running on it. The ESXi install is rather light and doing that remotely is no real headache. However, we also need to install customized servers on each one. Each of these servers needs to pull a lot of data during installation and installing at the facility makes no sense. Installing locally and pushing the image to the ESXi host at the facility is equally absurd. In our case, this could be a few weeks.

This is the situation we were faced with.

## The Solution

The solution was to build a single thumb drive that did a complete deployment of the ESXi host and all the virtual machines without touching the network.

Sounds simple or impossible depending on your understanding of ESXi. The answer is actually somewhere in between. It's incredibly difficult to figure out because even ESXi developers told me it wasn't possible. Once you figure it out, it's not horrible to repeat the process.

Basically:

1. Build the Virtual Machines to be put in the install
2. Export them to an OVA file
3. Put the OVA files on a web server
4. Put the installer on the thumb drive
5. Modify the installer to include the images
6. Set up first boot stuff
7. Create an image
8. Set up a thumb drive to deploy that image

Sounds easy, right? Sure...

## Creating the OVA

I'll breeze over this section. All you want to do is deploy a local ESXi server, install your virtual machines, and configure them as needed. If they're Linux boxes, then you will probably have a first boot script embedded. If it's a Windows server, don't forget the sysprep.

Once the virtual servers are ready, turn them off. Export them as an OVA. The OVF option will create a few files where the OVA will stick everything in one file.

Upload those files to a web server.

## ESXi on USB Thumb Drive

Our ESXi install disk needed extra drivers and we just used[ESXi Customizer][0]to get them in there. That tool doesn't really need any explanation.

When you have the ESXi ISO that you plan to use, you'll want to stick it on a USB stick. This is important because it lets you modify things easily. In my case, my thumb drive was sdc.

Prepping the thumb drive:

Putting files on the thumb drive:

Then modify some files for use with syslinux:

The ISO is no longer needed:

You can now unmount /mnt/usb if you want to verify that you can boot to it and run the installer. For the remainder, I'll assume that you left it since we have to still customize it.

## Creating Our Customizations: The Boot File

Later, we'll be creating ks.cfg and ovf-00.t00\. For the moment, assume that they exist. We need to tell ESXi about them.

Edit /mnt/usb/boot.cfg

In the modules line, we need to tell ESXi about ovf-00.t00\. Add /ovf-00.t00 to the list. It will look like the following.

Make sure to keep the ' --- ' between each item.

Where you see:

Add ks=usb:/KS.CFG:

Now the installer will know that it needs to load the /ovf-00.t00 file into memory and used the ks.cfg file as the kickstart when booting. If you don't quite see the magic yet, this already allows us to make the installation fully hands off.

## Creating Our Customizations: The Kickstart

Now we get to the fun part. The first step is to create a kickstart file. This is rather easy. It will look very much like this:

Edit /mnt/usb/ks.cfg:

There is a lot more you can with the kickstart, and ours is massive.

## Creating Our Customizations: The OVF

Notice in the above, I expect /ovf to exist. We're about to make it so! First we need to aquire ovftools. The VMware website has an[ovftool download][1].

Download the 64bit version and install it.

Now we need to make /ovf and the directories inside of it.

Take the installed ovftools and stick them in the tools directory.

We leave an empty files directory. This is where the post install part of the kickstart file will dump the OVA files.

Now we need to turn the ovf directory into ovf-00.t00\.

## Creating Our Customizations: Finished

Now that you have everything done, run umount /mnt/usb. Put that usb key into your server and boot to it. The installer will go through and do what we told it to.

Once it powers off, you're ready to make an image from the installed ESXi.

## Image Creation: USB Thumb Drive

For this part, we need yet another USB thumb drive. To keep this easy, just follow these basic steps.

1. Use a Windows box
2. Download[LiLi USB Creator][2]
3. Download[Clonezilla][3](I used the alternative stable)
4. Run LiLi
5. Tell it to install Clonezilla to a thumb drive

You'll probably want to make two copies of this. One for grabbing the image and one for pushing the image. For pushing the image, it needs to be at least as large as the image you're creating. For getting the image, 2GB is plenty.

Our ESXi image was large enough that we had to split up the thumb drive. A 64GB drive was given two partitions. The first was a 4GB FAT32 formatted partition that we placed Clonezilla on to. The second partition was Ext2 formatted and took up the remainder.

## Image Creation: Getting the Image

Boot to the plain Clonezilla USB drive on the ESXi server. You will probably want to also hook up a larger external hard drive for this.

1. Boot to Clonezilla (I usually ping the "Safe graphic" option)
2. optional: Plug in external hard drive
3. I usually ping the "Safe graphic" option
4. For the keymap settings, default is usually fine
5. Start\_Clonezilla
6. Choose device-image
7. Pick where you want the image to go (local\_dev for external HD)
8. Run through the settings for that option
9. Choose the Expert setting
10. Choose savedisk
11. Give it a name
12. You probably only have one option (sda) for the local disk
13. -q2 is usually best (partclone can handle sparse vmfs)
14. Make sure to select the -nogui option!!! (I usually deselect -c)
15. Compression is your choice
16. Since this is going to ext, feel free to make a large partition size
17. I usually choose to skip any checks (not safe)

For me, the command in the end looks something like this...

Now you get to wait for the image to get created. Once it's done, power off the box and we can start working on the deployment thumb drive.

## Image Creation: Clonezilla Modifications

We're going back to that extra flash drive you made. In our case, it was 64GB. The first 4GB partition (FAT32) has the standard Clonezilla image, for now. The remainder of the flash drive should be a single ext2 partition. You'll want yours to basically be the same, but maybe only a 16GB flash drive will be needed. This second partition will hold only one thing (your image).

We'll want to make some changes to the stuff LiLi put on the flash drive. For now, we'll assume the flash drive is sdb.

Now copy the image you created with Clonezilla into /mnt/sdb2 and unmount it.

That's all there is to getting our image on to the flash drive. That's not enough, though. Sure, we could launch the vanilla clonezill on here and tell it to use the image from the larger partition. However, we're trying to make this fully automatic. We came this far and we're finishing the task!

It's worth noting that live Linux distros use squashfs. This is an amazing tool because it lets you make a large file system into something very tiny, but this comes with the side effect that you're not able to actually modify the files. Instead, we have to take that file system, unpack it, repack it, and replace it.

To do this...

Edit /mnt/sdb1/syslinux/syslinux.cfg:

Feel free to copy/paste that exactly as is. This basically is what makes the interactive boot stuff in Clonezilla go away. It doesn't really do much more.

Moving on, we need to modify what happens after booting. You might remember this as a sleek set of options for what we want to do. We know what we want to do, so we're going to embed it.

First up is grabbing the existing squashfs image.

Now you have Clonezilla at your mercy. The first thing we want to do is burn the menu that launches when you start.

It's that simple. We delete that file. If you were to start up with only that change, you'd just end up at a shell. Now we need to tell it what to do when it boots up.

Edit etc/rc.local:

Edit usr/sbin/ocs-finished:

Make sure it's executable:

This assumes that you named your image BArK-Image, as I did. Case matters!

Now we need to create the new squashfs and put it on the flash drive.

That's all there is to it! Simple!

## Using It

Simply boot to it. Done. It'll deploy that image to the first disk.

## Final Thoughts

This is a very complicated process. As I stated before, it's for edge cases ONLY. If you're dealing with normal business needs, there are far superior options out there. For the other 0.03% out there, I really hope this can take you easily from the start to the end of this insane task.

[0]: http://esxi-customizer.v-front.de/
[1]: http://www.vmware.com/support/developer/ovf/
[2]: http://www.linuxliveusb.com/en/download
[3]: http://clonezilla.org/downloads.php