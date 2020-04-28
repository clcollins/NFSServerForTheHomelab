# Add shared filesystems to your homelab with an NFS server

A shared filesystem is a great way to add versatility and functionality to a homelab. A centralized filesystem shared among clients in the homelab makes organizing data and making backups considerably easier, and allows the clients to share data between them.  This is especially useful for both web applications load-balanced across multiple servers, and for persistent volumes used by Kubernetes, allowing pods to be spun up with persistent data on any number of nodes. Whether your homelab is made up of ordinary computers, surplus enterprise servers or Raspberry Pis or other Single Board Computers (SBCs), a shared filesystem is a useful asset, and an NFS server is a great way to create one.

I have written before about setting up our Private Cloud at Home, a homelab made up of Raspberry Pis or other SBCs, and maybe some other consumer hardware or a desktop PC.  An NFS server is an ideal way of sharing data between these components. As most SBCs like the Raspberry Pi run their operating system off of an SD card, there are some challenges. SD cards suffer from increased failures, especially being used as the OS disk for a computer, and they are not made to be constantly read from and written to. What we really need is a real hard drive: they are generally cheaper per gigabyte compared to SD cards, especially for larger disks, and they are less likely to sustain failures.  Raspberry Pi 4's now come with USB 3.0 ports, too, and USB 3.0 hard drives are ubiquitous and affordable. Its a perfect match, and for this example, we will use a 2TB USB 3.0 external hard drive plugged into a Raspberry Pi 4 running an NFS server.

Let's get started!

## Installing the NFS Server Software

In this example, we are running Fedora Server on a Raspberry Pi, but this can be done with other distributions as well. On Fedora, to run an NFS server we need the "nfs-utils" package, and lucky for us, it is already installed, at least in Fedora 31. We also need the rpcbind package if we are planning to run NFSv3 services, but it is not strictly required for NFSv4.

If for some reason these packages are not already on the system, install them with the `dnf` command:

```sh
# Intall nfs-utils and rpcbind
$ sudo dnf install nfs-utils rpcbind
```

Raspbian is another popular OS for use with Raspberry Pis, and the setup is almost exactly the same. The package names differ, but that is about the only major difference.  To install an NFS server on a system running Raspbian, we will need the following packages:

* nfs-common:  these are files common to both NFS servers and clients
* nfs-kernel-server:  the main NFS server software package

Raspbian uses `apt-get` for package management, and not `dnf` as Fedora does, so we will need to use that to install these packages:

```sh
# For a Raspbian system, use apt-get to install the NFS packages
$ sudo apt-get install nfs-common nfs-kernel-server
```

## Preparing a USB hard drive to use for the storage

As mentioned above, a USB hard drive is a good choice for providing the storage with Raspberry Pis or other SBCs, as the SD card used for the operating system disk image is not ideal. For our Private Cloud at Home, we will use cheap USB 3.0 hard drives for the large-scale storage.  After plugging the disk in, we can use `fdisk` to find out what device ID was assigned to it so we can work with it.

```sh
# Find your disk using fdisk
# Unrelated disk content omitted
$ sudo fdisk -l

Disk /dev/sda: 1.84 TiB, 2000398933504 bytes, 3907029167 sectors
Disk model: BUP Slim BK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe3345ae9

Device     Boot Start        End    Sectors  Size Id Type
/dev/sda1        2048 3907028991 3907026944  1.8T 83 Linux
```

In the example output above, the disks we are not interested in has been omitted for clarity. We can see the USB disk we want to use was assigned the device "/dev/sda", and we can see some information about the model (`Disc model: BUP Slim BK`), which helps identify the correct disk. We have a partition on the disk already, and by the size we can confirm it is the disk we are looking for.

*Note:* Make sure to identify the correct disk and partition for your device. It may be different than the example above.

Once we have identified the correct disk, we need to retrieve the UUID that it has been assigned to the partition on the disk. The UUID is used by the computer to make sure it is mounting the correct partition to the correct location, using the "/etc/fstab" config file.

We can retrieve the UUID of the partition using the `blkid` command:

```sh
# Get the block device attributes for the partition
# Make sure to use the partition that applies in your case.  It may differ.
$ sudo blkid /dev/sda1

/dev/sda1: LABEL="backup" UUID="bd44867c-447c-4f85-8dbf-dc6b9bc65c91" TYPE="xfs" PARTUUID="e3345ae9-01"
```

So in this case, the UUID of "/dev/sda1" is "bd44867c-447c-4f85-8dbf-dc6b9bc65c91". Yours will be different - make a note of this UUID.

## Configure the Raspberry Pi to mount this disk on startup, and then mount it

Now that we have identified the disk and partition we would like to use, we need to tell the computer how to mount it, to do so whenever it boots up, and go ahead and mount it now. Because this is a USB disk, and might be unplugged, we will also configure the Raspberry Pi to not wait on boot if the disk is not plugged in or is otherwise unavailable.

In Linux, this is done by adding the partition to the "/etc/fstab" configuration file, including where we want it to be mounted and some arguments to tell the computer how to treat it.  In this example, we are going to mount the partition to /srv/nfs, so start by creating that path:

```sh
# Create the mountpoint for the disk partition
$ sudo mkdir -p /srv/nfs
```

Next, we need to modify the "/etc/fstab" file.  The syntax of this file follows this format:

```txt
<disk id>     <mountpoint>      <filesystem type>     <options>     <fs_freq> <fs_passno>
```

In our case, we will use the UUID we identified earlier for the disk id.  As mentioned in the step above, the mountpoint we will use is "/srv/nfs". Usually, for the filesystem type, it would be best to select the actual filesystem, but since this will be a USB disk, we will use "auto".

For the options values, we will be using "nosuid,nodev,nofail".

_An aside:_ Man pages

That said, there are a _lot_ of possible options, and the Manual (man) pages are the best way to see what they are. Investigating the man page for for `fstab` is a good place to start:

```sh
# Open the man page for fstab
$ man fstab
```

This opens, in the terminal, the manual/documentation associated with the `fstab` command. In the man page, each of the options described is broken down, showing what they do and the common selections. Looking at "The fourth field (fs_mntopts)" gives us some basic information about the options that work in that field, and direct us to "man (8) mount" for more in-depth description of what the mount options are.  That makes sense, as the /etc/fstab file is, in essence, telling the computer how to automate mounting disks in the way we would manutally use the `mount` command.

We can get more information about the options we will be using from the mount man page. The number eight, in parentheses, is an indication of the section of the man page. In this case, section 8 is for "System Administration tools and Daemons".

Helpfully, you can get a list of the standard sections from the man page for `man`!

So, lets take a look at `man (8) mount`:

```sh
# Open Section 8 of the man pages for mount
$ man (8) mount
```

In this man page, we can examine what the options we are using actually do:

* nosuid: do not honor the suid/guid bit.  Do not allow any files that might be on the USB disk to be executed as root.  A good security practice.
* nodev: do not interpret character or block special devices on the file system; ie: do not honor any device nodes that might be on the USB disk.  Another good security practice.
* nofail: do not log any errors if the device does not exist.  This is a USB disk, and might not be plugged in, so ignore if that is the case.

Returning to the line we are adding to the "/etc/fstab" file, there are two final options: "fs_freq" and "fs_passno". The fs_freq and fs_passno values are related to somewhat legacy options, and _most_ modern systems will just use a "0" for both values, especially for filesystems on USB disks.  The fs_freq value has to do with the `dump` command, and making dumps of the filesystem.  The fs_passno value defines which filesystems to `fsck` on boot, and their order. If set, usually the root partition would be "1" and any other filesystems would be "2".  We will set the value to "0" to skip using `fsck` on this partition entirely.

In your preferred editor, open the "/etc/fstab" file and add the entry for the partition on the USB disk, replacing the values here with those gathered in the previous steps.

```sh
# With sudo, or as root, add the partition info to the /etc/fstab file
UUID="bd44867c-447c-4f85-8dbf-dc6b9bc65c91"    /srv/nfs    auto    nosuid,nodev,nofail,noatime 0 0
```

## Enabling and Starting the NFS Server

With the packages installed and the partition added to our "/etc/fstab" file, we can now go ahead and start the NFS server.  On a Fedora system, we need to enable and start two services, actually: rpcbind and nfs-server. We will use the `systemctl` command to accomplish this:

```sh
# Start NFS server and rpcbind
$ sudo systemctl enable rpcbind.service
$ sudo systemctl enable nfs-server.service
$ sudo systemctl start rpcbind.service
$ sudo systemctl start nfs-server.service
```

On Raspbian or other Debian-based distributions, we just need to enable and start the "nfs-kernel-server" service, and can do so the same way with the `systemctl` command.

### RPCBind

The rpcbind utility is used to map RPC services to ports on which they listen.  According to the rpcbind man page:

> When an RPC service is started, it tells rpcbind the address at which it is listening, and the RPC program numbers it is prepared to serve.  When a client wishes to make an RPC call to a given program number, it first contacts rpcbind on the server machine to determine the address where RPC requests should be sent.

In the case of an NFS server, rpcbind maps the protocol number for NFS to the port on which the NFS server is listening.  However, NFSv4 does not require the use of rpcbind. If we use *only* NFSv4, by removing versions two and three from the configuration, then rpcbind is not actually required anymore. It is included here for backward compatibility with NFSv3.

## Exporting the mounted filesystem

The NFS server decides which filesystems are shared with (exported to) which remote clients based on another configuration file, "/etc/exports". This file is just a map of host IPs (or subnets) to the filesystems to be shared, and some options (read-only or read-write, root squash, etc.).  The format of the file is:

```txt
<directory>     <host or hosts>(options)
```

In our example, we will be exporting the partition mounted to "/srv/nfs".  This is the "directory" piece.

The second part, the host or hosts, is the hosts we want to export this partition to. These hosts can be specified as a single host with a fully qualified domain name or hostname, or the IP address of the host, a number of hosts using wildcard characters to match domains (eg: *.example.org), IP networks (for example CIDR notation), or netgroups.

The third piece are options to apply to the export:

* ro/rw: export the filesystem as read-only or read-write
* wdelay: delay writes to the disk if another write is imminent, to improve performance (this is *probably* not as useful with a solid-state USB disk, if that is what you are using)
* root_squash: prevent any root users on the client from having root-access on the host, and set the root uid to "nfsnobody", a security precaution

For our example, we will test exporting the partition we have mouted at "/srv/nfs" to a single clinet - for example a laptop. In my case, I have identified the IP address for my laptop as 192.168.2.64 (your will likely be different).  We could share it to a large subent, but for testing we will limit it to the single IP address.  The CIDR notation for just this IP is 192.168.2.64/32 - a /32 subnet is just a single IP.

Using our preferred editor, we will edit the "/etc/exports" file with our directory, host CIDR, and the "rw" and "root_squash" options:

```txt
# Edit your /etc/exports file like so, substituting the information from your systems
/srv/nfs    192.168.2.64/32(rw,root_squash)
```

_Note:_ If you copied the "/etc/exports" file from another location, or otherwise overwrote the original with a copy, the SELinux context for the file may need to be restored.  This can be done with the `restorecon` command:

```sh
# Restore the SELinux context of the /etc/exports file
$ sudo restorecon /etc/exports
```

Once this is done, the NFS server just needs to be restarted to pick up the changes to the "/etc/exports" file:

```sh
# Restart the nfs server
$ sudo systemctl restart nfs-server.service
```

## Opening the Firewall for the NFS service

Some systems, by default, do not run a firewall service.  Raspbian, for example, defaults to open IPTables rules, with ports opened by different services immediately available from outside the machine.  Fedora server, by contrast, runs the firewalld service by default, and we must open the port for the NFS server (and rpcbind if we will be using NFSv3).  This can be done with the `firewall-cmd` command.

We can first check the zones used by firewalld, and get the default zone.  For Fedora Server, this will be the FedoraServer zone:

```sh
# List the zones
# Output omitted for brevity
$ sudo firewall-cmd --list-all-zones

# Retrieve just the default zone info
# Make a note of the default zone
$ sudo firewall-cmd --get-default-zone

# Permanently add the nfs service to the list of allowed ports
$ sudo firewall-cmd --add-service=nfs --permanent

# For NFSv3, we need to add a few more ports, nfsv3, rpc-mountd, rpc-bind
$ sudo firewall-cmd --add-service=(nfs3,mountd,rpc-bind)

# Check the services for the zone, substituting the default zone in use by your system
$ sudo firewall-cmd --list-services --zone=FedoraServer

# If all looks good, reload firewalld
$ sudo firewall-cmd --reload
```

And with that, we have successfully configured the NFS server with our mounted USB disk partition and exported it to our test system for sharing!  Now we can test mounting it on the system we added to the exports list.

## Testing the NFS exports

First, from the NFS server, create a file to read in the /srv/nfs directory:

```sh
# Create a test file to share
echo "Can you see this?" >> /srv/nfs/nfs_test
```

Now, on the client system we added to the exports list, first make sure the NFS client packages are installed.  On Fedora systems, this is the "nfs-utils" package, and can be installed with `dnf`.  Raspbian systems have the `libnfs-utils` package that can be installed with `apt-get`.

Install the NFS Client packages:

```sh
# Install the nfs-utils package with dnf
$ sudo dnf install nfs-utils
```

Once the client package is installed, we can test out the NFS export!  Again on the client, we can use the mount command with the IP of the NFS server and the path to the export and mount it to a location on the client, for this test, the "/mnt" directory. In this example, my NFS server's IP is 192.168.2.109, but yours will likely be different.

```sh
# Mount the export from the NFS server to the client host
# Make sure to substitute the information for your own hosts
$ sudo mount 192.168.2.109:/srv/nfs /mnt

# See if the nfs_test file is visible:
$ cat /srv/nfs/nfs_test
Can you see this?
```

Success!  With that, we now have a working NFS server for our homelab, ready to share files with multiple hosts, allow multi-read-write access, and provide centralized storage and backups for our data.  There are many options for shared storage for homelabs, but NFS is venerable, efficient, and a great option to add to our Private Cloud at Home homelab.  Further articles will expand on how to automatically mount NFS shares on clients, and how to use NFS as a storageClass for Kubernetes persistentVolumes.
