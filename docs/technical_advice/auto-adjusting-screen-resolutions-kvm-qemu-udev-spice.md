---
comments: true
---

# Auto-Adjusting Screen Resolutions in Virtual Machines: The Power of udev and SPICE

Have you ever been troubled by the non-responsive display resolution when working with virtual machines? If so, this article is tailor-made for you.

## The Problem

Using virtual machines, particularly with QEMU/KVM and `virt-manager`, I consistently encountered a snag: the screen resolution wouldn't automatically adjust to match the window size. This became particularly evident after upgrading a pre-built QEMU Virtual Machine. Everything worked perfectly at first, but the display resize feature stopped functioning later on, a problem I similarly noted with the netinstaller ISO.

Digging into this, I found the issue to be more prevalent with the XFCE desktop environment. Curiously, KDE and GNOME appeared unaffected, hinting at an underlying concern with XFCE. Considering the dated software on the pre-built VM, it's likely that updates or changes implemented between its initial release and subsequent upgrades introduced this issue.

Importantly, while my experience was with Kali Linux, this challenge is not exclusive to it; it's evident across multiple distributions. The silver lining? Utilizing `udev` rules can offer a comprehensive remedy for all Linux guest OSes that integrate `udev`.




## The Solution

Without further ado, here is the rule and the script that tackles the issue:


``` linuxconfig title="/etc/udev/rules.d/50-resize.rules"
ACTION=="change", KERNEL=="card0", SUBSYSTEM=="drm", RUN+="/usr/local/bin/resize"
```


??? note "Explaining the `udev` rule"
    This rule listens for any changes (`ACTION=="change"`) on the primary graphics card (`KERNEL=="card0"`) under the `drm` (Direct Rendering Manager) subsystem. The `drm` subsystem is responsible for interfacing with GPUs in Linux and handles tasks like rendering graphics and adjusting screen resolutions.

    Whenever a change is detected, it triggers (`RUN+`) our script (`/usr/local/bin/resize`), which will then adjust the screen resolution of the virtual machine.




``` bash title="/usr/local/bin/resize"
#!/bin/bash

declare -A unique_users display_to_user_mapping

# Identify unique users (excluding root)
for current_user in $(users); do
    [[ $current_user = root ]] && continue
    unique_users[$current_user]=1
done

# Map displays to users
for user in "${!unique_users[@]}"; do
    for display_info in $(ps e -u "$user" | grep -o 'DISPLAY=:[0-9]*'); do
        display_number="${display_info#*=}"
        display_to_user_mapping[$display_number]=$user
    done
done

# Adjust resolution for each user's display
for display in "${!display_to_user_mapping[@]}"; do
    associated_user="${display_to_user_mapping[$display]}"
    user_display="$display"
    output_device=$(sudo -u "$associated_user" DISPLAY="$user_display" xrandr | awk '/ connected/{print $1; exit;}')
    sudo -u "$associated_user" DISPLAY="$user_display" xrandr --output "$output_device" --auto
done
```

Dive deeper if you're curious about how the code functions:

??? note "How the script works"

    **Explanation**:

    - **User Identification**: The script begins by identifying all the users currently logged in, making a list of unique users. Notably, the `root` user is excluded. This is because in Linux environments, the `root` user typically does not run graphical sessions; it's generally considered insecure to do so. Hence, there's no screen to resize for the `root` user.

    - **Display Mapping**: Once the users are identified, the script maps the display environment variable (`DISPLAY`) to each user. This is crucial because in Linux, graphical sessions are linked with a `DISPLAY` value, which essentially indicates which screen or monitor the graphical output is being sent to.

    - **Resolution Adjustment**: With the user-to-display map in hand, the script then cycles through each display, determines its output device (like HDMI-1 or eDP-1) using the `xrandr` tool, and finally resizes the display to fit automatically using the `--auto` flag.


    **In a Nutshell**: The script automates what can be a manual and cumbersome process: ensuring your virtual machine's display dynamically adjusts to your window size, providing an uninterrupted user experience. By excluding `root`, it also ensures the security principle of minimizing potential attack surfaces is maintained.
    By pairing the above script with the simple udev rule, we can automatically adjust the display resolution whenever there's a change in the VM window size. Now, let's dive deeper.

## Understanding the Components

### SPICE

Simple Protocol for Independent Computing Environments (SPICE) is a remote display system primarily for virtualized environments. It improves the user experience by integrating audio, video, and several other features, offering a rich virtual machine console access experience.

### spice-agent

spice-agent is a guest agent that provides several facilities to improve the interaction between the host and guest VM, including clipboards syncing, automatic screen resolution adjustments, and more.

### udev

udev is a device manager for the Linux kernel. It dynamically manages device nodes in the /dev directory and offers userspace events whenever the status of hardware (like adding/removing devices) changes.

??? note "More about `udev`"

    `udev` is widely used in the Linux ecosystem. It has become the de facto standard for device detection and management in modern Linux distributions. Here's why it's prevalent:

    1. **Unified Device Management:** Before `udev`, device node handling in the `/dev` directory was static. As Linux systems became more dynamic, especially with the proliferation of USB devices and other hot-pluggable hardware, a more flexible system was required. `udev` fills this need by providing dynamic device node creation and management.

    2. **Part of `systemd`:** In many modern Linux distributions, `udev` functionality has been merged into `systemd` as `systemd-udevd`. `systemd` itself is a widely adopted init system and service manager for Linux. This integration further solidified `udev`'s position in the Linux ecosystem.

    3. **Configurability:** `udev` allows administrators and developers to write rules specific to their needs. This level of customizability is beneficial for various applications, ranging from desktops to servers to embedded systems.

    4. **Consistent Device Naming:** `udev` provides persistent device naming, ensuring that devices get consistent names across reboots. This feature is essential for administrators and users, ensuring, for example, that storage devices always get mounted at the expected locations.

    5. **Hotplug Support:** Modern computers frequently have devices added or removed while the system is running (think USB devices, SD cards, etc.). `udev` supports such hotplug scenarios, ensuring the system reacts appropriately when devices are plugged or unplugged.

    6. **Compatibility:** While `udev` provides advanced features and configurability, it's designed to be backward compatible. This means even systems or setups that depended on the older, static `/dev` approach can still function under `udev`.

    Given its importance and the functionalities it offers, `udev` is a critical component in nearly all major Linux distributions today. Whether you're on a desktop Linux distribution like Ubuntu, Fedora, or Debian, or server distributions like CentOS or RHEL, or even embedded systems, `udev` (or its functionalities as part of `systemd-udevd`) plays a vital role in device management.


### udev rules

These are configurations that dictate how udev responds to specific hardware changes. In our context, we utilize udev rules to trigger our screen resolution adjustment script when there's a change in the display settings of the VM.

## About the `SPICE Agent`
For the auto-resize feature to work with QEMU/KVM when using the SPICE protocol, you typically need the `spice-vdagent` (often referred to as the "SPICE agent") running inside the guest. The SPICE agent facilitates many enhanced functionalities, including clipboard sharing, automatic screen resizing, and more.

Here's how it all comes together:

1. **Guest Display Resizing:** When you resize the QEMU/KVM guest window on the host, the SPICE client (like `virt-viewer` or `spicy`) sends a message to the SPICE server component running with QEMU, indicating the new screen dimensions.
    
2. **SPICE Server to Guest:** The SPICE server then communicates this information to the `spice-vdagent` running inside the guest. This is achieved using a virtual channel.
    
3. **Notification to the System:** Once the `spice-vdagent` in the guest receives the resize information, it triggers a change event in the graphics subsystem. This event is detected by the udev subsystem thanks to the rule you set up, which then launches the `/usr/local/bin/resize` script to adjust the screen resolution accordingly.
    

To ensure the SPICE agent is operational:

1. **Installation:** Ensure `spice-vdagent` is installed in the guest.
	```bash
	sudo apt-get install spice-vdagent
	```
    
2. **Autostart:** Ensure the `spice-vdagent` service is enabled to start automatically on boot. Depending on the init system, this might involve using `systemctl` or another service manager.
    
3. **Manual Start:** You can also manually start the agent from within the guest, usually by just running `spice-vdagent` from the terminal.
    

If the SPICE agent isn't running, the screen resize event from the SPICE client won't be propagated to the guest system, and as a result, the udev rule won't be triggered.

## Wrap-Up

Harnessing the capabilities of `udev`, `shell scripting`, `SPICE`, and `spice-agent`, we've fashioned a solution to tackle the irksome issue of non-responsive VM screen resolutions. No more manual tinkering — just seamless auto-adjustments.

## Credits

The insights and techniques mentioned in this article have been sourced from a few excellent discussions and solutions from the community:

- **Finding Sessions as Root**: [How to get the list of all active X sessions and owners of them?](https://unix.stackexchange.com/questions/117083/how-to-get-the-list-of-all-active-x-sessions-and-owners-of-them) on Unix Stack Exchange.
    
- **Resizing via udev**:
    
    - [No auto resize with SPICE and virt-manager](https://superuser.com/questions/1183834/no-auto-resize-with-spice-and-virt-manager) on Super User.
    - [How to fix can't resize Kali Linux VM screen display via Virt Viewer running on Proxmox VE (PVE) with default XFCE4 desktop environment?](https://dannyda.com/2020/10/22/how-to-fix-cant-resize-kali-linux-vm-screen-display-via-virt-viewer-running-on-proxmox-ve-pve-with-default-xfce4-desktop-environment/#1-Via-Command-directly-(Resize-Manually)) by Dannyda.com.





Got thoughts, or better ways to handle this? Let's discuss below or connect on [LinkedIn](https://www.linkedin.com/in/dani-speh-7221b9220/)!

> **Note**: When deploying the script, ensure to accompany it with the required udev rule, grant the script execute permissions, and have `spice-agent` running in your VM.

Happy virtualizing! 🖥️🔄🖼️


