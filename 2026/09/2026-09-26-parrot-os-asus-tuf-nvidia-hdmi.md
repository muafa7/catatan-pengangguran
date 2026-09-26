# Parrot OS on ASUS TUF F15/F16: NVIDIA, Wayland, HDMI, and SQUASHFS

I tried installing Parrot OS on my ASUS TUF laptop, and the main pain point turned out to be NVIDIA, HDMI, and a broken installation attempt.

The laptop uses hybrid graphics with Intel + NVIDIA, so I expected graphics to be the part most likely to cause problems.

## Testing the Live USB

Before installing Parrot OS, I booted into the Live USB first to check whether the hardware worked properly.

Most things seemed fine:

- Wi-Fi worked
- keyboard worked
- touchpad worked
- internal display worked
- desktop itself was usable

Then I plugged in HDMI.

The system froze.

Since everything else seemed fine, I decided not to spend too much time debugging the Live environment.

I continued with a full installation and planned to fix HDMI afterward.

## Installation Failed with SQUASHFS Errors

During the first installation attempt, I suddenly started getting a lot of errors like:

```text
SQUASHFS error: Unable to read ...
SQUASHFS error: Unable to read page ...
```

The errors kept appearing repeatedly.

At around the same time, KDE also broke and showed this message:

```text
The screen locker is broken and unlocking is not possible anymore.

In order to unlock it, switch to a virtual terminal
(e.g. Ctrl+Alt+F2), log in to your account and execute the command:

loginctl unlock-session 4

Then log out of the virtual session by pressing Ctrl+D,
and switch back to the running session (Ctrl+Alt+F8).
```

So the broken screen locker happened during the same failed installation attempt where the SQUASHFS errors appeared.

It was not a separate NVIDIA or HDMI problem.

At first, the errors looked bad enough that I thought the problem might be:

- SSD
- RAM
- filesystem
- Parrot OS itself
- USB installer

Before debugging anything deeper, I decided to reflash the Parrot ISO to the USB drive.

Then I tried installing again.

The second installation succeeded.

The SQUASHFS errors disappeared, and the broken screen locker problem did not happen again.

So in my case, the most likely cause was the installation media or a bad USB write.

## SQUASHFS Troubleshooting

SQUASHFS is commonly used for compressed read-only filesystem images in Linux installation media.

If the installer cannot reliably read it, possible causes include:

- corrupted ISO
- bad USB write
- problematic USB drive
- problematic USB port
- installation media corruption
- less commonly, RAM or other hardware issues

In my case, reflashing the USB fixed it.

So instead of immediately assuming:

```text
SSD is broken
```

a better troubleshooting order is:

```text
verify ISO
    ↓
reflash USB
    ↓
try installation again
    ↓
change USB port if needed
    ↓
try another USB drive if needed
    ↓
investigate hardware if the problem continues
```

Lesson learned:

> If a Linux installer starts throwing a lot of SQUASHFS errors, try reflashing the installation media before debugging the machine itself.

## Back to the HDMI Problem

After Parrot OS was successfully installed, HDMI still needed to be fixed.

Since the laptop uses both Intel and NVIDIA graphics, I started checking the NVIDIA driver setup.

The rough process was:

```text
disable Nouveau
    ↓
install NVIDIA proprietary driver
    ↓
configure NVIDIA DRM for Wayland
    ↓
rebuild initramfs
    ↓
reboot
    ↓
verify NVIDIA
    ↓
test HDMI
```

## Disable Nouveau

Parrot may initially use Nouveau, the open-source NVIDIA driver.

I disabled it before using the proprietary NVIDIA driver.

Create:

```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

Add:

```conf
blacklist nouveau
options nouveau modeset=0
alias nouveau off
```

Or directly:

```bash
echo -e "blacklist nouveau\noptions nouveau modeset=0\nalias nouveau off" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
```

## Install NVIDIA Driver

Update the package list:

```bash
sudo apt update
```

Install the NVIDIA driver:

```bash
sudo apt install nvidia-driver
```

Install `nvidia-smi`:

```bash
sudo apt install nvidia-smi
```

`nvidia-smi` is useful to verify whether the NVIDIA driver is actually loaded and working.

## Configure NVIDIA for Wayland

Since the desktop session uses Wayland, NVIDIA DRM modesetting needs to be enabled.

Create or edit:

```bash
sudo nano /etc/modprobe.d/nvidia.conf
```

Add:

```conf
options nvidia-drm modeset=1
options nvidia-drm fbdev=1
```

Depending on the driver and kernel version, `modeset=1` may already be enabled by default.

I explicitly configured it while troubleshooting HDMI.

## Rebuild initramfs

After changing kernel module configuration, rebuild initramfs:

```bash
sudo update-initramfs -u
```

Then reboot:

```bash
sudo reboot
```

## Verify NVIDIA

After rebooting, check NVIDIA:

```bash
nvidia-smi
```

If everything works, it should show information such as:

- NVIDIA GPU
- driver version
- VRAM usage
- GPU utilization
- running GPU processes

Check NVIDIA DRM modesetting:

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

Expected:

```text
Y
```

Check framebuffer support:

```bash
cat /sys/module/nvidia_drm/parameters/fbdev
```

Expected:

```text
Y
```

Check loaded GPU modules:

```bash
lsmod | grep -E "nvidia|nouveau"
```

Ideally NVIDIA modules should be loaded and Nouveau should not be active.

Example:

```text
nvidia
nvidia_modeset
nvidia_drm
nvidia_uvm
```

## Check the Current Session

Check whether KDE is running on Wayland:

```bash
echo $XDG_SESSION_TYPE
```

Expected:

```text
wayland
```

## Check Display Detection

KDE provides `kscreen-doctor`, which is useful for checking detected displays.

Run:

```bash
kscreen-doctor -o
```

This helps check whether HDMI is actually detected even when the external monitor shows nothing.

## Why HDMI Can Behave Differently

This laptop uses hybrid graphics:

```text
Intel iGPU
    +
NVIDIA dGPU
```

The internal laptop display and HDMI output do not necessarily use the same GPU path.

So it is possible to have:

```text
Internal Display
    ↓
works normally

HDMI
    ↓
uses NVIDIA path
    ↓
driver / DRM / GPU configuration problem
    ↓
freeze or no display
```

That explains why Parrot OS could work normally on the laptop screen while HDMI caused problems.

## Useful NVIDIA Debugging Commands

Check NVIDIA:

```bash
nvidia-smi
```

Check loaded NVIDIA and Nouveau modules:

```bash
lsmod | grep -E "nvidia|nouveau"
```

Check detected GPU devices:

```bash
lspci | grep -Ei "vga|3d|display"
```

Check NVIDIA kernel logs:

```bash
sudo dmesg | grep -i nvidia
```

Check DRM logs:

```bash
sudo dmesg | grep -i drm
```

Check HDMI-related logs:

```bash
sudo dmesg | grep -i hdmi
```

Or check everything together:

```bash
sudo dmesg | grep -Ei "nvidia|nouveau|drm|hdmi"
```

Check session type:

```bash
echo $XDG_SESSION_TYPE
```

Check display outputs:

```bash
kscreen-doctor -o
```

Check NVIDIA DRM modesetting:

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

Check NVIDIA framebuffer:

```bash
cat /sys/module/nvidia_drm/parameters/fbdev
```

Check whether Nouveau is loaded:

```bash
lsmod | grep nouveau
```

If nothing is returned, Nouveau is not loaded.

Check NVIDIA modules:

```bash
lsmod | grep nvidia
```

## Troubleshooting Order

If I run into the same NVIDIA or HDMI problem again, I would troubleshoot it in this order.

### 1. Check detected GPUs

```bash
lspci | grep -Ei "vga|3d|display"
```

### 2. Check loaded drivers

```bash
lsmod | grep -E "nvidia|nouveau"
```

### 3. Check NVIDIA

```bash
nvidia-smi
```

### 4. Check whether the session uses Wayland

```bash
echo $XDG_SESSION_TYPE
```

### 5. Check NVIDIA DRM

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

and:

```bash
cat /sys/module/nvidia_drm/parameters/fbdev
```

### 6. Check detected displays

```bash
kscreen-doctor -o
```

### 7. Check logs

```bash
sudo dmesg | grep -Ei "nvidia|nouveau|drm|hdmi"
```

### 8. Rebuild initramfs if configuration changed

```bash
sudo update-initramfs -u
```

### 9. Reboot

```bash
sudo reboot
```

### 10. Test HDMI again

Plug the HDMI cable back in and check:

```bash
kscreen-doctor -o
```

## What I Learned

The annoying part about this installation was not really installing Parrot OS itself.

It was everything around installation media, NVIDIA, hybrid graphics, Wayland, and external displays.

Things worth remembering:

- Test the Live USB before committing to a full installation.
- A working internal display does not mean HDMI will work.
- External display outputs can use a different GPU path.
- Hybrid Intel + NVIDIA laptops make Linux graphics troubleshooting more complicated.
- SQUASHFS errors do not automatically mean the SSD is broken.
- Reflash the installation USB before investigating deeper hardware problems.
- A bad USB write can produce very scary-looking errors.
- The broken KDE screen locker happened during the same SQUASHFS installation failure.
- Nouveau and the proprietary NVIDIA driver should not both be trying to control the GPU.
- `nvidia-smi` is one of the first commands to check after installing NVIDIA drivers.
- NVIDIA DRM modesetting matters when using Wayland.
- Kernel module configuration changes usually require rebuilding initramfs.
- `kscreen-doctor -o` is useful for checking whether KDE actually sees the external monitor.
- `dmesg` is useful when the GUI gives no useful information.
- Always save the commands while debugging.

That last one is the reason this note exists.
