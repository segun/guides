#Part 1: Setting up and starting an NFS Server

##What is NFS

A Network File System (NFS) allows remote hosts to mount file systems over a network and interact with those file systems as though they are mounted locally. In other words It is a client/server system that allows users to access files across a network and treat them as if they resided in a local file directory. This guide will instruct you on how to install, setup and run NFS Server on CentOS 7.

#### NOTE

>The steps in this guide require root privileges. Be sure to run the steps below as **root** or with the `sudo` prefix.

##Prerequisites

####Step 1: Install nfs-utils

	yum install nfs-utils	

####Step 2: Enable and start services

	systemctl enable rpcbind
	systemctl enable nfs-server
	systemctl enable nfs-lock
	systemctl enable nfs-idmap
	systemctl start rpcbind
	systemctl start nfs-server
	systemctl start nfs-lock
	systemctl start nfs-idmap

####Step 3: Firewall settings

	firewall-cmd --permanent --zone=public --add-service=nfs
	firewall-cmd --permanent --zone=public --add-service=mountd
	firewall-cmd --permanent --zone=public --add-service=rpc-bind
	firewall-cmd --reload

##Setting up NFS Server

To setup an NFS Server, you need to edit one configuration file. 

	/etc/exports

This configuration file contains the directories you want to make available to the clients and the IP Address or DNS address of the clients. In very large installations with lots of clients, you can use a network and a netmask. Finally you can also provide wild-cards in extreme cases. All these will be covered below.

It is worthy to note that there are two other files that you can edit to give your NFS Server a more secure outlook, these are 

	/etc/hosts.allow 
	/etc/hosts.deny 

These two files specify which clients on the network can use services on your the server. Each line of the file contains a single entry listing a service and a set of clients. These files are discussed in details in the second part of this guide. **Securing your NFS Server**

###The Exports File (`/etc/exports`)
This file contains a list of entries; each entry indicates a volume that is shared and how it is shared. The description here is minim although the description will suffice for most NFS Setups.

An entry in `/etc/exports` will typically look like this:

	path client1(optionC11,optionC12) client2(optionC21,optionC22)

#####Where

`path`

> is the path that you want to make available on the network. It can also an entire volume. The path is shared recursively, meaning if you share a path, then all paths under it shared as well.

`client1 and client2`

> are the client machines that will have access to `path`. As stated earlier, The client machines may be listed by their DNS address or their IP address (e.g., `client1.company.com` or `192.168.1.8`). Using IP addresses is more reliable and more secure. A DNS Address can change it's IP and you might be sharing the `path` with some other client you didn't intend.

`optionXX`

> is the option listing for each client machine. It describes what kind of access the client machine will have. Some of the important options are:

>> `ro`: The directory is shared read only; the client machine will not be able to write to it. This is the default.

>> `rw`: The client machine will have read and write access to the directory.

>> `no_root_squash`: By default, any file request made by user root on the client machine is treated as if it is made by user nobody on the server. (Excatly which UID (User ID) the request is mapped to depends on the UID of user "nobody" on the server, not the client.) If `no_root_squash` option is specified, then root on the client machine will have the same level of access to the files on the system as root on the server. This can have serious security implications, although it may be necessary if you want to perform any administrative work on the client machine that involves the paths. **You should not specify this option without a good reason.**

`no_subtree_check`: If a large volume is shared, specifying this option speeds up the file transfers

There are other options, please refer to the manual page for exportfs for more options

	man exportfs
	
##Examples

####Step 1: Edit /etc/exports.

Let's assume you have two clients with IP addresses 192.168.1.7 and 192.168.1.8 and you want to share your home directory (with all sub-paths in home directory) with first client and only the home directory of a user called john with the second client, your `/etc/exports` file will look like this.

	/home			192.168.1.7(ro)
	/home/john		192.168.1.8(rw)

This will make /home available to client with IP Address 192.168.1.7 in read-only mode and /home/john available to client with IP Address 192.168.1.8 in read-write mode.

Let's assume you also want to make /home available to client with IP Address 192.168.1.8, your `/etc/exports` file will look like this

	/home			192.168.1.7(ro)	192.168.1.8(ro)
	/home/john		192.168.1.8(rw)
	
For a large installation with a lot of clients, you can use a network and a netmask to supply a range of IP Addresses.

	/home		192.168.1.0/255.255.255.0(ro)
	
This will make /home available to all clients within the IP Range 192.168.1.0 and 192.168.1.255.

Finally, you can use a wild card `*` instead of IP Addresses. For example
	
	/home/john				192.168.*(rw)
	/usr/lib/src			*.example.com(ro)
	/usr/local/lib/src		*(rw,no_root_squash,no_subtree_check)


####Step 2: Restart NFS Server

	systemctl restart nfs-server
	
####Step 3: Connect to the NFS Server from client.

##### Create the mount point

	mkdir -p /mnt/nfs/home
	
##### Mount the NFS Path

	mount -t nfs 192.168.1.100:/home /mnt/nfs/home/

######Where 
	192.168.1.100 is the server IP Address
	/home is the path being shared on the server.
	
##### Check to make sure the path is mounted

	df -k

You should see something like

	192.168.0.100:/home           19G   33M   19G   1% /mnt/nfs/home
	
#### Step 4: Make the mount permanent.

Using the above steps, the mount is lost if the client machine is rebooted. To make the mount persist over reboots, you must edit `/etc/fstab` and add the below line.

	192.168.1.100:/home    /mnt/nfs/home   nfs defaults 0 0

Cheers, you have successfully configured NFS Server one CentOS 7 and connected to it. But this configuration is in-secure, Part 2 of this guide will deal with how to secure your NFS Server.
