# Setup Raspberry PI

Guide how to install Raspberry Pi OS on sd with an external drive for data.

## Create ssh key

Create an ssh key with this command and select the filename for it.
```sh
ssh-keygen -t rsa
```
With this key, you can connect via ssh.

## Install OS

1. Download and run raspberry pi imager.
2. Select `Raspberry PI OS (64-bits)`.
3. Choose storage.
4. Write it.
5. Boot raspberry pi.

## Install OS on SSD

1. Open raspberry pi imager.
2. Select `Raspberry PI OS Lite (64-bits)`.
3. Configure it with:
    - Set username and password.
    - Override locale settings if needed.
    - Disable telemetry.
4. Choose storage (ssd).
5. Write it.
6. In raspberry pi imager select `USB boot` as OS and write it to storage.
7. Resize root partition on boot using `fdisk`/`cfdisk`.


> NOTE:
> In my case rpi didn't make new user and imaged default system.
> I will setuped it by myself

### Setting up

Copy public ssh key
```sh
ssh-copyid user@n.n.n.n
```

Uncomment or add this line to `/etc/ssh/ssh_config`
```
PasswordAuthentication no
```

### Configure deploy user

Create a user for ansible with sudo-rights.

```sh
sudo adduser username
sudo passwd username
sudo adduser username sudo
sudo mkhomedir_helper username
```

Copy `.ssh/authorized_keys` to the user's home dir.

## Configure drive

### Make filesystem

1. Connect the drive by USB and find its drive name (ex. `ls /dev/sd*`).
2. Make a partition with 
    ```sh
    sudo cfdisk /dev/sda
    ```
    - Choose the `gpt` partition table.
    - Create partition(s).
        - for boot (`/boot`, change type to 11, use begin and end sectors like in SD-card)
        - for data (`/data`, not home)
        - for OS (`/`)
        - for swap (for this partition change type)

3. Make `ext4` filesystem (for `/` and `/data`):
    ```sh
    sudo mkfs.ext4 /dev/sda1
    ```

### Add to fstab

1. Find UUID of a partition with:
    ```sh
    sudo lsblk /dev/sda1 -f
    ```
2. Create a folder for data `/data`.
3. Add a record to `/etc/fstab`:
    ```fstab
    UUID=12345678-9abc-defghujkl-mnopqrstuxyz   /data   ext4    defaults,noatime    0   2
    ```
4. Remount drives with `sudo mount -a`.
5. Create a folder for the user:
    ```sh
    sudo mkdir /data/username
    sudo chown username:username /data/username
    ln -s /data/username /home/username/data
    ```
