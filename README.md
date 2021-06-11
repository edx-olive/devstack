# Campus.IL Devstack

This repository contains an edX devstack configuration that closely matches the configuration of israelxedu.gov.il,
and automatically sets up the Campus IL development branches.

This configuration is mainly useful for:
* Testing backported changes against the Campus IL configuration locally
* Developing the Campus IL theme

## Usage instructions

1. Clone this repository:

```terminal
git clone git@gitlab.com:opencraft/client/campus/campus-devstack.git
```

2. Provision the devstack:

```terminal
vagrant plugin install vagrant-vbguest --plugin-version 0.21
vagrant up
```

3. Once provisioned, SSH in and run the services:

```terminal
vagrant ssh
sudo -Hu edxapp bash
paver run_all_servers
```

## Troubleshooting

### NFS isn't working, or the edx-platform checkout/npm install steps hang forever

NFS problems are extremely common with when using the edX Devstack. It often helps to use vboxsf shares instead.

To try using vboxsf, reload the VM with `VAGRANT_USE_VBOXFS` set:
```terminal
VAGRANT_USE_VBOXFS=true vagrant reload --provision
```

### The VirtualBox Guest Additions failed to install, or vboxsf isn't loaded

If using vboxsf with Vagrant 2.0.0 with VirtualBox 5.2.0 or similar, you may see the VirtualBox Guest Additions
5.2.0 fail to compile, and/or a vboxsf mount failure:
```
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel modules.
VirtualBox Guest Additions: Look at /var/log/vboxadd-setup.log to find out what went wrong
VirtualBox Guest Additions: Running kernel modules will not be replaced until the system is restarted
VirtualBox Guest Additions: Starting.
VirtualBox Guest Additions: modprobe vboxsf failed
An error occurred during installation of VirtualBox Guest Additions 5.2.0. Some functionality may not work as intended.
In most cases it is OK that the "Window System drivers" installation failed.

...

mount -t vboxsf -o uid=0,gid=33 edx_src /edx/src

The error output from the command was:

: No such device
```

This happens due to an incompatibility between Vagrant and VirtualBox, and can be worked around by manually installing
The VirtualBox Guest Additions:
```terminal
vagrant ssh 
wget http://download.virtualbox.org/virtualbox/5.2.0/VBoxGuestAdditions_5.2.0.iso
sudo mount -o loop VBoxGuestAdditions_5.2.0.iso /mnt

# This should succeed at building the modules against the kernel, but it'll still fail to load the vboxsf module
# This is fine and expected, just reboot the VM with Vagrant afterwards and it'll boot with the new 5.2.0 vboxsf module
sudo sh -x /mnt/VBoxLinuxAdditions.run
exit
```

One completed, reboot the VM and the newly built modules should load up automatically:
```terminal
VAGRANT_USE_VBOXFS=true vagrant reload --provision
```

### MongoDB didn't shut down gracefully, or `[Errno 111] Connection refused`

This applies when you're presented with a Django error page containing a connection failure relating to pymongo:
```
Exception Type: ConnectionFailure
Exception Value: [Errno 111] Connection refused
Exception Location: /edx/app/edxapp/venvs/edxapp/local/lib/python2.7/site-packages/pymongo/mongo_client.py in __init__, line 369
```

This means MongoDB is unreachable - usually the reason for this is that it never started up due to the mongod.lock file never
being removed. This virtually always happens when the VM is forcibly halted (e.g. `vagrant halt --force`), or if MongoDB
doesn't otherwise have a chance to gracefully shut down.

The solution is to manually remove the lock file and start MongoDB up again:
```
vagrant ssh
sudo rm /edx/var/mongo/mongodb/mongod.lock

# (Optionally) repair any corrupt collections that may have resulted from the MongoDB not properly shutting down
sudo -u mongodb mongod --dbpath /edx/var/mongo/mongodb --repair --repairpath /edx/var/mongo/mongodb

sudo start mongodb
```

## Notes

### Vagrant base box

The currently used base box is a Campus specialized box which contains the Campus related fixes and git cherry-picks. This means that it will work for the Campus devstack, but cannot be used for other projects.

### Configuration branch

The configuration branch (`gabor/fix-box-creation`) of the devstack **differs** from stage/prod configuration branch and shouldn't be changed to a branch which is not based on that (`gabor/fix-box-creation`).

As of the upstream ginkgo Vagrant devstack configuration is broken we had to fix that without affecting stage/production configuration. See the related [comments](https://github.com/edx-olive/configuration/pull/22).

The special configuration branch contains fixes to be able to build the base and Campus devstack properly.
