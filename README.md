# virtnbdbackup-docker

*"Backup utility for libvirt, using the latest changed block tracking features. Create thin provisioned full and incremental backups of your kvm/qemu virtual machines."*

(Refer to the [original source code](https://github.com/abbbi/virtnbdbackup) first, for better understanding, get familiar with syntax and help.)

## Overwiew:
Virtnbdbackup-docker is intended for scenarios where isn't viable for SysAdmins to provide a up-to-date dependencies (stable distros); or when this is totally impossible due to system constraints (inmutable / embedded rootfs, docker oriented OSes, etc.)

This image includes 'virtnbdbackup' and 'virtnbdrestore' utils installed along with required dependecies, and currently is being built from `debian:bullseye-slim` as base.

It has been successfully tested on UnRaid v6.9.2, but should work the same on many other distros, as much as below requirements can be accomplished.

## Requirements:
- Docker Engine. See [Docker Documentation](https://docs.docker.com/get-docker/) for further instructions
- libvirt >=6.0.0
- To have performed the punctual modifications on VM's XML file and image format, as pointed at source code [README](https://github.com/abbbi/virtnbdbackup/blob/master/README.md), so this tool will work for you.

Note: This image carries latest 'qemu-utils' as of its base OS for internal processing of images during restoration.

## Bind mounts:

- Virtnbdbackup needs to access libvirt's socket in order to work correctly, and attempts this via `/var/run/libvirt` path.

  In basically all mainstream distros of today (Debian, RedHat, Archlinux and the countless distros based on these) as in this image, `/var/run` is a symlink to `/run` and `/var/lock` a symlink to `run/lock`.
  For the vast majority of scenarios the correct bind mount is: `-v /run:/run`

  But in certain cases (e.g. UnRaid) `/run` and `/var/run` are different folders. Under this scenario you need to bind mount with `-v /var/run:/run`
  And most likely, also with either `-v /var/lock:/run/lock` or `-v /var/run/lock:/run/lock` in order to run this container correctly.

  If you're in trouble with this, *read source FAQ* and create a persistent container (as described below) in order to debug, and get the correct bind mounts that work for your main host (you're encouraged to commit to improve this image.)

- Virtnbdbackup and virtnbdrestore create sockets for backup/restoration jobs tasks at `/var/tmp`. Ensure to always add a bind mount with `-v /var/tmp:/var/tmp`

- Finally, to warrant clearness with all input commands, it's convenient to use same paths for backup (and restoration) bind mounts at both endpoints, such as `-v /mnt/backups:/mnt/backups` in order to parse commands in same way as you were running it natively on your main host.

## Usage Examples:

### Full Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdbackup -d <domain-name> -l full -o /mnt/backups/<domain-name>`


### Incremental Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdbackup -d <domain-name> -l inc -o /mnt/backups/<domain-name>`


### Restoration of Backup:


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdrestore -i /mnt/backups/<domain-backup> -a restore -o /mnt/restored`


Where `/mnt/restored` is an example folder in your system, where virtnbdrestore will rebuild virtual disk(s) based on existing backups, with its internal block device name, such as 'sda', 'vda', 'hdc', etc.

### Interactive mode / debugging virtnbdbackup:

You can also run the container in interactive mode by running its build in shell, and then execute multiple backup/restoration commands, as needed. This also very is useful for debugging purposes:


`docker run -rm -it \`

`-v /var/tmp:/var/tmp -v /run:/run -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored' \`

`adrianparilli/virtnbdbackup-docker \`

`/bin/bash`


and execute commands as desired. The container will keep running until you type `exit` on the internal shell.

### Persistent container:
In the above examples, the container will be removed as soon the invoked command has been executed. This is the optimal behaviour when you intend to automatize operations,  such as incremental backups. In addition, you can set a persistent container with all necessary bind mounts with:


`docker create --name <container-name> \`

`-v /var/tmp:/var/tmp -v /run:/run -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored' \`

`adrianparilli/virtnbdbackup-docker \`

`/bin/bash`


Just creating a new container (with custom name) with mount points set and ready to run in interactive mode. To start it and automatically enter into the internal shell, just type:


`docker start -i <container-name>`


And again, stopping it with the command `exit` from its shell.

For more usage examples (including latest features) refer to source code [README](https://github.com/abbbi/virtnbdbackup/blob/master/README.md)

## Quick Notes for SysAdmins:

- Modifications on VM's XML files while domains are running, requires to restart such domains.
- Restoration jobs are independent of domain's state (it can be running or not) but actual restoration of domain has to be done by hand, by:
  - Stopping the domain
  - Renaming / replacing image files on its final location
  - Starting the domain
- Newest versions of virtnbdbackup don't require to start a domain each time you need to perform an incremental backup, but this is still required in order to create a new backup chain.
