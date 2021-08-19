# virtnbdbackup-docker

*"Backup utility for libvirt, using the latest changed block tracking features. Create thin provisioned full and incremental backups of your kvm/qemu virtual machines."*

(Refer to the [original source code](https://github.com/abbbi/virtnbdbackup) first, for better understanding, get familiar with syntax and help.)

## Overwiew:
Virtnbdbackup-docker is intended for scenarios where isn't viable for SysAdmins to provide a full python3 environment plus up-to-date dependencies (old distros); or when this is totally impossible due to system constraints (inmutable / embedded rootfs, docker oriented OSes, etc.)

This image includes 'virtnbdbackup' and 'virtnbdrestore' utils installed, along with required dependecies, and currently is being built from `debian:bullseye-slim` as base.

By now, it has been successfully ested on UnRaid, and should work the same on many other distros as much as below requirements can be accomplished.

## Requirements:
- Docker Engine. See [Docker Documentation](https://docs.docker.com/get-docker/) for further instructions
- libvirt >=6.0.0 (Qemu version seems to be not important, since this image carries 'qemu-utils' for internal processing.)
- To have performed the punctual modifications on VM's XML file as pointed in [source code's README](https://github.com/abbbi/virtnbdbackup), so this tool will work for you.

## The key it's to determine correct bind mounts:

- Virtnbdbackup needs to access libvirt's socket in order to work correctly, and attempts this via `/var/run/libvirt` path.

  Basically in all mainstream distros of nowadays (Debian, RedHat, Archlinux and the countless distros based on these) `/var/run` is a symlink to `/run` and `/var/lock` a symlink to `run/lock`, so the correct bind mount in vast majority of scenarios should be: `-v /run:/run`.

  But in certain cases (such as UnRaid) `/run` and `/var/run` are different folders. In this scenario you need to bind mount with `-v /var/run:/run`, and sometimes also with `-v /var/lock:/run/lock`) in order to run this container correctly.

  If none of this work for you, *read source FAQ* create a persistent contiainer as described below, in order to debug the behaviour from inside and get the correct bind mounts that work for your main host.

- Virtnbdbackup creates sockets for backup/restoration tasks at /var/tmp. Ensure to mimic this with `-v /var/tmp:/var/tmp` always

- Finally, for clearness with all the commands you will input, it's convenient to mimic backup and restoration bind mounts at both endpoints, such as `-v /mnt/backups:/mnt/backups` and so on (for virtnbdrestore, most likely you will need two bind mounts, one for backups location, and another for the path where VM will be restored)

## Usage Examples:

### Full Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`docker-virtnbdbackup \`

`virtnbdbackup -d <domain-name> -l full -o /mnt/-backups/<domain-name>`


### Incremental Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`docker-virtnbdbackup \`

`virtnbdbackup -d <domain-name> -l inc -o /mnt/-backups/<domain-name>`


### Restoration of Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored \`

`docker-virtnbdbackup \`

`virtnbdrestore -i /mnt/-backups/<domain-backup> -a restore -o /mnt/restored`


Where `/mnt/restored` is just an example of folder in your system where virtnbdrestore will rebuild virtual disk(s) present on backups, since will create images with names corresponding with its internal block device name, such as 'hdc'.

### Persistent container:
In above examples, container will be removed as soon the invoked command has been executed. This is the optimal behaviour when you intend to automatize operations (such as incremental backups.)

In addition, you can set a persistent container with all necessary bind mounts with:

`docker create --name <container-name>`

`-v /var/tmp:/var/tmp -v /run:/run -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored' \`

`docker-virtnbdbackup \`

`/bin/bash`

And attach to its Shell with: `docker start -i <container-name>` to perform manual backups/restorations, or for debugging purposes.

## Quick Notes for SysAdmins:

- Modifications on VM's XML files while these are running, require to restart the domain.
- Backups will be executed on running domains only.
- Restoration is independent of domain's state (it can be running or not) but domain restoration -if needed- has to be done by hand, such as stopping the domain and renaming /replacing image files on its final location.
