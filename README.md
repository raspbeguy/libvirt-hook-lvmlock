# Libvirt hook for lvmlock management

This script is meant to be used as hook by libvirt in order to automatically manage locks on a LVM shared volume group.

## A bit of explanation

### Use case

If you have a cluster of libvirt hosts, you need at least one shared storage pool containing the images of your domains. One way to achieve it is by having a shared LVM volume group, which is a volume group with a lock manager, `lvmlockd`, which will be in charge of allowing or denying to LVM access to the logical volumes in it. This is useful to prevent concurrent access from multiple host and thus keeping volumes integrity by not booting multiple times from different locations. Indeed, classic filesystems do not support concurrent read/write access.

For more information about shared storage, you can read [my blog entry (in french)](https://hashtagueule.fr/posts/la-virtu-pour-les-nuls-le-stockage) I wrote a few years ago.

### How a shared volume group works

The lock manager `lvmlockd` keeps track of *who* currently uses *what*, *somehow*.

* By *who*, I mean a host in your cluster.
* By *what*, I mean a logical volume.
* By *somehow*, I mean a lock backend.

Lock backends options are `sanlock` and `dlm`.

* `sanlock` uses the shared storage itself (a small dedicated logical volume) to store lock ownerships and types.
* `dlm` uses network communication between hosts and also needs an actual cluster manager.

No matter which backend you use (even though `sanlock` is more reliable and easier to setup IMHO), they provide to `lvmlockd` a source of truth for lock management.

So basically, a host has to ask for a lock on a logical volume prior to using it. It should also release this lock when it doesn't need the logical volume anymore. Without a lock, the host isn't allowed to access the logical volume.

A lock has a type, which can be either:
* exclusive: when obtained, no other host is allowed to request another lock, no matter which type, on the logical volume.
* shared: when obtained, other hosts are allowed to request other shared locks on the logical volume but are still not allowed to request exclusive locks.

Exclusive locks are supposed to be used in a very short time period when multiple hosts have no option to have access to the same logical volume, for instance in case of a live migration. At any time, a host can request a mutation of an exclusive lock it owns to a shared lock. It can also request the opposite, but only when no other host owns any shared lock on the logical volume.

### Application in a libvirt context

* Before running a domain, a host must request an exclusive lock on all of the logical volume used by the domain.
* After shutting it down, it must release its locks.
* Before a live migration, the source host must mutate its exlusive locks to shared locks and the target host must request matching shared locks.
* After the live migration, the source host must release its shared locks and the target host must mutate its shared locks to exclusive locks.

## Installation

Just copy or symlink the script `qemu` and its config file `qemu.ini` in `/etc/libvirt/hooks/`. This is a hook script, so libvirt will execute it on a given list of events in the domains lifecycle. See [libvirt qemu hook documentation](https://libvirt.org/hooks.html#etc-libvirt-hooks-qemu) for more info about the hook events.

## Configuration

The `log` section is pretty self-explanatory.

The `actions` section defines how to match hook events with the three actions you can do on a logical volume lock: `exclusive`, `shared`, `unlock`.

## Limitations

Sadly, this hook script doesn't make everything possible (yet). Indeed two other hook events are needed in the case of a live migration: the source host should trigger an event to let the script mutate the exclusive lock into a shared lock, and the target host another event when source host has finished releasing all resources to be able to mutte its lock as an exclusive lock. Everything else is covered, but without this extra event, there is no other way than manually running `lvchange -asy <domain_logical_volume>` on the source host before any live migration, and `lvchange -an <domain_logical_volume>` on the target host after.

I opened [an issue](https://gitlab.com/libvirt/libvirt/-/issues/37).

## Contribution

I gladly accept any help for making this code better, including making it cleaner and adding features. Also if you are wanting to work on the libvirt issue I mentionned earlier, please do! The community behind libvirt is quite busy and they also are welcoming any help they can get.
