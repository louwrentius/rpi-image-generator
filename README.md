# ansible-raspberry-pi

Create custom Raspberry Pi images based on your Ansible inventory. A custom image will be generated for each host configured in your inventory. Once the Pi boots from the image, it will be immediately operational and accessible over the network through SSH.

Ansible-raspberry-pi aims to fully automate the process of generating a custom Raspberry Pi image, including the step of writing
the image to SD card. This means that once the playbook is done, you remove the SD card from the reader and stick it into the Pi. When you turn on the Pi, it will be fully operational and accessible through SSH.

This playbook generates a custom OS image based on the vanilla OS image:

1. Hostname and networking is configured
2. SSH access is enabled
3. A local user is created with public SSH key for secure remote access
4. Optional: Software is installed as per your personal preferences
5. Optional: security patches are applied 
6. The image is written to SD card
7. Optional: once the SD card is inserted and the Pi has booted, post-image tasks are run

I use it myself to provision a user including public SSH key so I can immediately use the device after first boot from the SSD card, without any manual configuration, usually required for vanilla Ubuntu or Raspbian OS images.

This is ideal if you have to provision a lot of Raspberry Pis, or if you often want to reset the OS install of a Pi back to a clean state. The playbook in its current form can't generate and write images in parallel, the process is sequential.

### Image Template
The playbook generates a custom base image from which all unique images are derived. This makes it possible to perform an apt upgrade on the template, so all new images generated from this base template won't need to perform any updates after deployment.
 
**Important**
I've written this for myself and thus it is only tested on MacOS. 

This playbook expects that you have access to a Linux machine (virtual machine or physical computer) on
which the RPi images are created and edited. You need 10+ GB storage available depending on how many images you need to generate.

### Networking
By default, a static IP address is configured for the Pi based on the ansible_host IP address in the inventory.
If you need to configure the Pi with DHCP, you have to manually edit the 'create-template' role and change the appropriate template for network configuration. The 'files' folder of this role contains rc.local files with an option to show the
Pi's IP-address as part of the login screen at the console, which can help determining the IP-address when DHCP is used.

# Process Overview
This is a general overview of what this playbook provides:

1. Downloads Ubuntu RPI image (validates checksum)
2. Generates a template image and adds custom settings shared by all Pis (includes package update!)
3. Generates an individual image based on the template image for each Pi
4. Writes the generated image with DD over SSH on the local SD card.
5. Plays a sound when ready (MacOS only)

# Preparation / requirements

1. Determine the device name of the SDCARD reader/writer (Like: /dev/sdf or /dev/disk11 on MacOS)
2. Update the inventory/shared/authorized_keys file with the appropriate public ssh key for the ansible user
3. You need an existing computer or virtual machine with Linux installed and sufficient free space (at leat 10GB)
4. It is advised to use an SSD to host the working_directory to speed up the image generation process

# MacOS users
You must run the playbook with sudo because writing the image to the SD card requires elevated privileges.

# Extra parameters

### force_template_update (bool)

By default, if an image template already exists, it is not touched. To update/recreate it, either delete the file or use this option.
The existing template will be overwritten with a fresh copy of the vanila image and recreated with all customisation.

This option will force existing images for individual hosts to also be regenerated from the template.

### force_template_patch_update (bool)

This will perform an apt update on the template image. Keeping the template up-to-date means that if you deploy multiple
devices, they are all up-to-date after deployment and won't have to update 50+ security updates. 

### skip_template_copy (bool)

By default, if the template is updated, the entire image file will be overwritten with a fresh copy of the vanila image.
This parameter assures that the existing template will *not* be overwritten, just mounted and edited to reflect the changes in the roles/playbooks/variables.

### force_image_update (bool)

By default, if an image has been generated previously, this image will be written to the SD card. If you want to generate a new image due to
configuration changes, specify this parameter to force creation of a new image.

### Manually mount image

    losetup --find --partscan --show <imagefile.img>

Based on the output - like '/dev/loop0' - mount the appropriate device:

mount /dev/loop0p2 /mnt/folder 

Now you can inspect the image contents. You can also start a chroot at /mount/folder and manually test commands as if you are running the computer based on this template for debugging purposes.

### How images are customised

First a copy is made of the vanilla OS image, this copy will be the template from which all individual host images are derived.
This copy is mounted on a loop device and the appropriate partition of the image is then mounted. From this moment out, files
can be added or changed. 

The more complex step is to install software on the image, or to run software updates. Running an 'apt upgrade' on the template image is very usefull as all derived images will immediately be up-to-date and individual Raspberry Pis don't have to run
lengthy 'apt upgrade' steps.

**Chroot**
To run the update process, we use 'chroot', which is like running the actual image as if we are running a Raspberry Pi.
On Macs with ARM-based chips, it's possible to run an ARM-version of Raspian or Ubuntu. The role that generates the template, also copies the qemu-aarch64-static binary so we can emulate ARM on x86(-64) as well, although this is slower.
We use Chroot also to install additional software inside the image.

