---
layout: post
title:  "Flash flatcar to computer using USB stick"
date:   2023-09-12 18:00:00 +0200
categories: flatcar nuc infrastructure boot
---

After spending some time having a look to [flatcar documentation][flatcar-documentation] I realised that it was not so clear (at least for me) how to install flatcar in a computer using a USB stick. So the goal is documenting the procedure from zero to running flatcar in a computer.

Below steps will cover how to download and copy ISO image to a USB stick using a Linux distribution.

## Requirements
* 2 USB sticks
* Docker
* dd

## Download ISO image

This is a very simple step. Just go to the official site and [download][download-flatcar-iso] the stable ISO.

## Copy ISO image to USB stick

In my case, path `/dev/sdc` was assigned to the USB stick so I run below command to copy ISO image to USB.

```
dd if=flatcar_production_iso_image.iso of=/dev/sdc bs=1M
```

Bear in mind that path `/dev/sdc` should be replaced by the one assigned to your USB stick.

## Generate configuration file

By default, flatcar OS is not configured to allow logins so I needed to create a configuration file to provision a user to be able to login.

To do this, there is some tools that will be used.

1. Create a yaml file similar to the below one
    ```
    variant: flatcar
    version: 1.0.0
    passwd:
      users:
        - name: pi
          password_hash: ""
          ssh_authorized_keys:
            - xxxx
          home_dir: /home/pi
          groups:
            - wheel
          shell: /bin/bash
    ```
2. I wanted to be able to login to flatcar using SSH key as well as password (remove `password_hash` if password is not required). To generate a password hash, it can be done using `mkpasswd`. 
    ```
    docker run -ti --rm quay.io/coreos/mkpasswd --method=yescrypt
    ```
3. Run below command to generate the configuration file using a docker container where butane is already configured 
    ```
    cat config.yaml | docker run --rm -i quay.io/coreos/butane:latest > ignition.json
    ```

After generating the configuration file, I needed to copy it to a different USB stick because the one I copied flatcar ISO image is formatted in read only mode ([ISO9660][iso9660])

## Flash flatcar to computer

I am not going to document how to boot from USB stick because it will look different depending on the BIOS of the computer.

In my case, after booting from USB stick I noticed my computer disk had several partitions. What I did is to use `fdisk` to delete partitions and format the whole disk.

Configuration file was stored in a second USB stick that was assigned to path `/dev/sdc`. That is why I specify a path for the ignition file different than current folder. To mount the second USB stick to be able to use the ignition configuration file, I ran below command
```
sudo mount /dev/sdc1 /mnt
```

After formatting the whole disk and mounting the second USB stick with the configuration file, it was time to flash flatcar. It is done using below command:

```
flatcar-install -d /dev/sda -i /mnt/ignition.json
```

I set the device to install flatcar to `dev/sda` because it was the mount point assigned by the OS to the main disk.

[flatcar-documentation]: https://www.flatcar.org/docs/latest/installing/bare-metal/installing-to-disk/
[download-flatcar-iso]: https://www.flatcar.org/docs/latest/installing/bare-metal/booting-with-iso/
[iso9660]: https://en.wikipedia.org/wiki/ISO_9660