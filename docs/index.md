# Welcome to MkDocs

For full documentation visit [mkdocs.org](https://www.mkdocs.org).

## Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.


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

