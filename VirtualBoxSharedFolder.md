# Mounting a VirtualBox Shared Folder in Ubuntu

This tutorial shows you how to mount a VirtualBox shared folder in an Ubuntu virtual machine to a specific location, and make it persistent across reboots.

## Step 1: Create the Shared Folder in VirtualBox

1. Right-click your virtual machine in VirtualBox and choose 'Settings.'
2. Click 'Shared Folders,' then the '+' symbol.
3. Choose the folder on your host machine to be shared, name it "VM_Shared," and select any other options you need.
4. Uncheck the 'Auto-mount' checkbox and click 'OK.'

## Step 2: Create the Mount Point in Ubuntu

Open a terminal in your Ubuntu VM and run:

```bash
mkdir ~/Share
```

## Step 3: Unmount Any Existing Mounts

If the folder is already mounted elsewhere, unmount it:

```bash
sudo umount /media/sf_VM_Shared
```

## Step 4: Mount the Shared Folder

Mount the shared folder to your desired location:

```bash
sudo mount -t vboxsf VM_Shared ~/Share
```

## Step 5: Make the Mount Permanent

Edit the `/etc/fstab` file to make the mount permanent:

```bash
sudo nano /etc/fstab
```

Add the following line at the end, replacing `your_username` with your actual username:

```
VM_Shared /home/your_username/Share vboxsf defaults 0 0
```

Save and exit the text editor.

## Step 6: Test the Configuration

Test the new configuration without restarting:

```bash
sudo mount -a
```

## Step 7: Reboot the Virtual Machine

Reboot to ensure the shared folder mounts correctly:

```bash
sudo reboot
```

## Completion

You should now have the shared folder mounted to `~/Share` every time you boot the virtual machine. Enjoy accessing your shared files!
