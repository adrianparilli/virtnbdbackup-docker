# virtnbdbackup-docker

*"Backup utility for libvirt, using the latest changed block tracking features. Create thin provisioned full and incremental backups of your kvm/qemu virtual machines."*

## Overwiew
Virtnbdbackup-docker is intended for scenarios where isn't viable to provide the necessary environment, such as dependencies or tools, due to system limitations; such as an old OS version, inmutable or embedded rootfs, live distros, docker oriented OSes, etc.

For production usage on servers or hosts without these mentioned limitations, it's much better to deploy directly, via your package manager or directly downloading/installing the latest version of its [source code](https://github.com/abbbi/virtnbdbackup)

It was originally made to be used on UnRaid (tested since v6.9.2), but should work equally fine on any other GNN/Linux distro, as much as below requirements can be accomplished.

This image includes 'virtnbdbackup' and 'virtnbdrestore' utils installed along with their required dependecies and other utilities such as latest Qemu Utils and OpenSSH Client in order to leverage all available features.

Currently, is being built from `debian:bookworm-slim` as base OS.

Refer to the [source code](https://github.com/abbbi/virtnbdbackup) for better understanding, and get familiar with syntax, help and troubleshooting.

## Requirements
- Docker Engine on the host server. See [Docker Documentation](https://docs.docker.com/get-docker/) for further instructions
- libvirt >=v6.0.0. on the host server, but >=v7.6.0 is highly recommended to avoid [patching XML VM definitions](https://github.com/abbbi/virtnbdbackup#libvirt-versions--760-debian-bullseye-ubuntu-20x)
- Qemu guest agent installed and running inside guest OS. For *NIX guests, use the latest version (as of named) available from the package manager. For Windows guests, install latest [VirtIO drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/)

## Bind mounts

- Virtnbdbackup needs to access libvirt's socket in order to work correctly, and attempts this via `/var/run/libvirt` path.

  In basically all mainstream distros of today (Debian, RedHat, Archlinux and the countless distros based on these) as in this image, `/var/run` is a symlink to `/run` and `/var/lock` a symlink to `run/lock`.
  Therefore, for the vast majority of scenarios the correct bind mount is: `-v /run:/run`

  But in some operating systems, `/run` and `/var/run` are still separated folders. Under this scenario you need to bind mount with `-v /var/run:/run`
  And most likely, you will need to mount with either `-v /var/lock:/run/lock` or `-v /var/run/lock:/run/lock` in order to run this container correctly.

  If you're in trouble with this, read source [FAQ](https://github.com/abbbi/virtnbdbackup#faq) and create a [persistent container](#persistent-container) in order to debug, and get the correct bind mounts that work for your main host.

- Virtnbdbackup and virtnbdrestore create sockets for backup/restoration jobs tasks at `/var/tmp`. Ensure to always add a bind mount with `-v /var/tmp:/var/tmp`

- Finally, to warrant clearness with all input commands, it's convenient to use same paths for backup (and restoration) bind mounts at both endpoints, such as `-v /mnt/backups:/mnt/backups` in order to parse commands in same way as you were running it natively on your main host.

## Usage Examples

### Full Backup


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdbackup -d <domain-name> -l full -o /mnt/backups/<domain-name>`


### Incremental Backup


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdbackup -d <domain-name> -l inc -o /mnt/backups/<domain-name>`


### Backup Restoration


`docker run --rm \`

`-v /run:/run -v /var/tmp:/var/tmp -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored -v /etc/libvirt/qemu/nvram:/etc/libvirt/qemu/nvram \`

`adrianparilli/virtnbdbackup-docker \`

`virtnbdrestore -i /mnt/backups/<domain-backup> -a restore -o /mnt/restored`


Where `/mnt/restored` is an example folder in your system, where virtnbdrestore will rebuild virtual disk(s) based on existing backups, with its internal block device name, such as 'sda', 'vda', 'hdc', etc.

Mount point `/etc/libvirt/qemu/nvram` is required when involved backup includes NVRAM disks (e.g. UEFI Operating Systems), since virtnbdresore will attempt to restore it to its original location.

### Interactive mode / debugging virtnbdbackup

You can also run the container in interactive mode by running its build in shell, and then execute multiple backup/restoration commands, as needed. This also very is useful for debugging purposes:


`docker run -rm -it \`

`-v /var/tmp:/var/tmp -v /run:/run -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored' \`

`adrianparilli/virtnbdbackup-docker \`

`/bin/bash`


and execute commands as desired. The container will keep running until you type `exit` on the internal shell.

### Persistent container
In the above examples, the container will be removed as soon the invoked command has been executed. This is the optimal behaviour when you intend to automatize operations,  such as incremental backups. In addition, you can set a persistent container with all necessary bind mounts with:


`docker create --name <container-name> \`

`-v /var/tmp:/var/tmp -v /run:/run -v /mnt/backups:/mnt/backups -v /mnt/restored:/mnt/restored' \`

`adrianparilli/virtnbdbackup-docker \`

`/bin/bash`


Just creating a new container (with custom name) with mount points set and ready to run in interactive mode. To start it and automatically enter into the internal shell, just type:


`docker start -i <container-name>`


And again, stopping it with the command `exit` from its shell.

## Quick Notes for SysAdmins

- When libvirt <= 7.6.0, modifications on VM's XML files to enable incremental backup capability can be made while domains are running, but requires to restart such domains for changes take effect.
- Only a 'full' backup chain operation requires to start the domain in advance. All other operations (copy, diff, inc) doesn't need the domain running.
- Both 'full and 'inc' checkpoints created while domain is running are stored in memory, but only saved to qcow images as bitmaps when domain is shut down. Under OS or libvirt failing scenarios (e.g. power drops, system crashes, etc.) non-saved checkpoints are lost, resulting into broken backup chains that can't receive more incremental checkpoints. This is due to Qemus Bitmap Persistence's way of working and more details can be found [here.](https://qemu-project.gitlab.io/qemu/interop/bitmaps.html#id17) Involved backups can be normally restored, though.
- Restoration task is independent of domain's state, but actual domain restoring has to be done by hand, by:
  - Stopping the domain
  - Renaming / replacing image files on its final location
  - Starting the domain
  Files as persistent NVRAMs are automatically restored
- Ensure to read and understand documentation regarding virtnbdbackup completely

Any pull request to improve this work is more than welcome!
