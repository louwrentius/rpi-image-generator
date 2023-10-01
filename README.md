# rpi-image-generator
This Ansible-based tool allows you to generate custom Raspberry Pi images similar to [pi-gen][pigen] but it 
also automatically writes images to SD card or any other storage device. You have to decide if pi-gen may be
a better fit for your needs or this tool could be of help.

It's good to know that this tool generates a template image from which all individual images are derived.
Patches can be applied to this template image, so individual hosts are up-to-date at first-time boot.

[pigen]: https://github.com/RPi-Distro/pi-gen

Imagine that you need to deploy 10 Raspberry Pis, this is what your workflow will look like:

1. insert (first) SD card in card reader
2. run playbook
3. create image for pi and writes it to SD card
- Your computer plays a voice to inform you that the SD card is ready
4. remove the finished SD card 
5. insert a new SD card

![console](demo01)

[demo01]: https://github.com/louwrentius/rpi-image-generator/blob/33b682834e8025c2c9ff66b93b3b3c14a347cf8b/nextcard.png

The playbook loops from step 3 forward untill all hosts have their custom image written to SD card.

# Preparation / requirements
1. You need a virtual machine or physical host on which you can generate the Raspberry Pi images (20GB+ free)
1. Update the inventory/shared/authorized_keys file with the appropriate public ssh key for the ansible user
1. Update the inventory/inventory file with the correct host names and IP-addresses
1. Determine the device name of the SDCARD reader/writer (Like: /dev/sdf or /dev/disk11 on MacOS)
1. you need ansible installed

# Configuring the environment
Edit the following files in the group_vars/all directory as required:
1. vars.yml
2. network-settings.yml 
3. Edit either debian.yml or ubuntu.yml to select the image you want
4. Copy the public SSH key into inventory/shared/authorized_keys

# Ansible Playbook Examples

## Configuring the image-generator host
The image-generator host is a Linux-based virtual machine or physical host on which we generate the custom images. 
We prepare this host like this:

    ansible-playbook -i inventory/inventory playbooks/configure-linux-vm.yml -b 

This playbook creates some folders (as configured in the inventory) downloads the OS image file, and generates the 
template image. The template image will be setup with additional software and patches will be installed.

- If you run this playbook on Linux, you don't need a separate virtual machine or physical host. You can make your local
host the image-generator host, but I've never tested this myself.

## Generate custom images and write them to SD card

When this command is run, an image will be generated for each specified Pi in the inventory. It will also be written
to SD card. 

    sudo ansible-playbook -i inventory/inventory -b playbooks/generate-and-deploy-image.yml 

The playbook will wait until the SD card is removed. Then it waits until a new SD card is insered, and the image for the next host will be written.

This playbook also plays a sound on MacOS so you know the SD card is ready.

## Extreme Automation
The end-to-end playbook automates the entire processs, from configuring the image-generator host, downloading the OS image,
generating a template and building the individual Pi images. It also writes those images to SD card.

    sudo ansible-playbook -i inventory/inventory playbooks/end-to-end.yml -b 

Please note that this is a bit slow as for every image written to SD card, the image-generator host is configured again,
even though most steps can be skipped after the first time.

# Detailed background
This is a general overview of what this playbook provides:
1. Downloads Ubuntu RPI image (validates checksum)
2. Generates a template image and adds custom settings shared by all Pis (includes package update!)
3. Generates an individual image based on the template image for each Pi
4. Writes the generated image with DD over SSH on the local SD card.
5. Plays a sound when ready (MacOS only)

# Image customisation overview
1. Hostname and networking is configured
2. SSH access is enabled
3. A local user is created with public SSH key for secure remote access
4. Optional: Software is installed as per your personal preferences
5. Optional: security patches are applied # this is the default
6. The image is written to SD card
7. Optional: once the SD card is inserted and the Pi has booted, post-image tasks are run

# MacOS users
You must run the playbooks that write images to SD card with *sudo* because
writing the image to the SD card requires elevated privileges.

# Limitations

- I've written this for myself as an fun excersize and I've only tested this on MacOS
- You need a virtual machine or physical host with ~3GB+ per Pi of storage space.
- You can't use the ansible-playbook --limit parameter because the code doesn't take the --limit parameter into acount.
It's recommended to comment out all hosts you don't want to deploy in the inventory file. 
- The playbook can't generate and write images in parallel, the process is sequential.

## Networking
By default, a static IP address is configured for the Pi based on the ansible_host IP address in the inventory.
If you need to configure the Pi with DHCP, you have to manually edit the 'create-template' role and change the appropriate template for network configuration. 

The 'files' folder of the 'create-template' role contains rc.local files with an commented (disabled) option to show the
Pi's IP-address as part of the login screen at the console, which can help determining the IP-address when DHCP is used.

# Extra parameters

## force_template_update (bool)
By default, if an image template already exists, it is not touched. To update/recreate it, either delete the file or use this option. The existing template will be overwritten with a fresh copy of the vanila image and recreated with all customisation.
This option will force existing images for individual hosts to also be regenerated from the template.

## force_template_patch_update (bool)
This will perform an apt update on the existing template image. Keeping the template up-to-date means that if you deploy multiple
devices, they are all up-to-date after deployment and won't have to run updates individually. 

## skip_template_copy (bool)
By default, if the template is updated, the entire image file will be overwritten with a fresh copy of the vanila image.
This parameter assures that the existing template will *not* be overwritten, just mounted and edited to reflect the changes in the roles/playbooks/variables.

## force_image_update (bool)
By default, if an individual Pi image has been generated previously, this image will be written to the SD card. If you want to generate a new image due to configuration changes, specify this parameter to force creation of a new image.


## How images are customised
First a copy is made of the vanilla OS image, this copy will be the template from which all individual host images are derived.
This copy is mounted on a loop device and the appropriate partition of the image is then mounted. From this moment out, files
can be added or changed. 

The more complex step is to install software on the image, or to run software updates. Running an 'apt upgrade' on the template image is very usefull as all derived images will immediately be up-to-date and individual Raspberry Pis don't have to run
lengthy 'apt upgrade' steps.

**Chroot**
To run the update process, we use 'chroot', which is like running the actual image as if we are running a Raspberry Pi.
The role that generates the template, also copies the qemu-aarch64-static binary so we can emulate ARM on x86(-64). 
On Macs with ARM-based chips, it's possible to run an ARM-version of Raspian or Ubuntu and the static binary is not used.  
We also use chroot to install additional software on the template image.

## Manually mount image
These steps are executed on the host that acts as the image-generator.

    losetup --find --partscan --show <imagefile.img>

Based on the output - like '/dev/loop0' - mount the appropriate device:

mount /dev/loop0p2 /mnt/folder 

Now you can inspect the image contents. You can also start a chroot at /mount/folder and manually test commands as if you are running the computer based on this template for debugging purposes.

Once the image is unmounted, you can clear all loop devices with:

    losetup -D
