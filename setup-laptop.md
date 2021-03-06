# setup-laptop.sh


## Read User Input
```sh
read -p "Username: " USER
```


## Security
Setup a firewall:
```sh
pacman --quiet --sync --needed --noconfirm ufw
ufw enable
```


## Video
```sh
pacman  --quiet --sync --needed --noconfirm \
        adobe-source-code-pro-fonts \
        arandr \
        autorandr \
        awesome \
        intel-media-driver \
        libvdpau-va-gl \
        mesa \
        picom \
        vulkan-icd-loader \
        vulkan-intel \
        xbindkeys \
        xf86-video-nouveau \
        xorg-server \
        xorg-xinit \
        xorg-xrandr \
        xorg-xrdb \
        xscreensaver \
        xterm
```

```sh
cat << 'EOF' >> /home/${USER}/.profile
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
  exec startx > /tmp/x.log 2>&1
fi
EOF
```

```sh
cat << 'EOF' > /home/${USER}/.xinitrc
#!/bin/sh
if [ -f ~/.Xresources ]; then
  xrdb -merge ~/.Xresources
fi
if [ -f ~/.xbindkeysrc ] && ! pgrep --euid "${USER}" xbindkeys > /dev/null; then
  xbindkeys
fi
picom &
xscreensaver -no-splash &
screen_layout &
exec awesome
EOF
```

```sh
cat << 'EOF' > /home/${USER}/.Xresources
*VT100*foreground: gray90
*VT100*background: black
*VT100*faceName: Source Code Pro:size=12:antialias=true
EOF
```

```sh
cat << 'EOF' > /home/${USER}/.inputrc
set bell-style none
EOF
```

```sh
cat << 'EOF' > /home/${USER}/.xbindkeysrc
"/home/${USER}/bin/screen_layout"
    m:0x40 + c:33
    Mod4 + p

"xscreensaver-command --lock"
    m:0x50 + c:67
    Mod2+Mod4 + F1
EOF
```

```sh
cat << 'EOF' > /home/${USER}/bin/screen_layout
#!/bin/sh

autorandr --change
~/.fehbg
EOF

chmod +x /home/${USER}/bin/screen_layout
```

Configure Awesome Window Manager:
```sh
mkdir --parents "/home/${USER}/.config/awesome"

cp /etc/xdg/picom.conf "/home/${USER}/.config"
cp /etc/xdg/awesome/rc.lua "/home/${USER}/.config/awesome"
cp /usr/share/awesome/themes/zenburn/theme.lua "/home/${USER}/.config/awesome"

sed --in-place 's/awful\.layout\.layouts\[.*\]/awful\.layout\.layouts\[3]/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/^terminal/s/xterm/uxterm/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/^beautiful.init/s/get_themes_dir/get_xdg_config_home/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/^beautiful.init/s/default/awesome/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/titlebars_enabled =/s/true/false/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/"p"/s/modkey/modkey, "Control"/' "/home/${USER}/.config/awesome/rc.lua"
sed --in-place '/screen = awful.screen.preferred,/a size_hints_honor = false,' "/home/${USER}/.config/awesome/rc.lua"

sed --in-place '/^theme\.border_width/s/2/0/' "/home/${USER}/.config/awesome/theme.lua"
sed --in-place '/^theme\.font/s/8/10/' "/home/${USER}/.config/awesome/theme.lua"

echo "opacity-rule = [ \"80:class_g = 'UXTerm'\" ];" >> /home/${USER}/.config/picom.conf
```

### Manual Configuration

To create a profile for a certain screen layout:

1. Use `arandr` to configure a screen layout.
2. After replacing `<NAME>` with a suitable name, run:

        autorandr --skip-options crtc --save <NAME>


## Buttons and Power Management

Setup power management:
```sh
pacman --quiet --sync --needed --noconfirm acpid tlp hdparm

systemctl enable acpid
systemctl start acpid

systemctl enable tlp
systemctl start tlp
```

Handle volume and mute buttons:
```sh
mkdir -p /etc/acpi/handlers

cat << 'EOF' > /etc/acpi/events/audio
event=button/(mute|volume.*)
action=/etc/acpi/handlers/audio.sh %e
EOF

cat << 'EOF' > /etc/acpi/handlers/audio.sh
#!/bin/sh

pulseaudio_pid=$(pgrep pulseaudio)
user_name=$(ps --format user --no-headers $pulseaudio_pid)
user_id=$(id --user $user_name)

case $2 in
  MUTE)  sudo --user $user_name XDG_RUNTIME_DIR=/run/user/$user_id pactl set-sink-mute @DEFAULT_SINK@ toggle ;;
  VOLDN) sudo --user $user_name XDG_RUNTIME_DIR=/run/user/$user_id sh -c "pactl set-sink-mute @DEFAULT_SINK@ false; pactl set-sink-volume @DEFAULT_SINK@ -5%" ;;
  VOLUP) sudo --user $user_name XDG_RUNTIME_DIR=/run/user/$user_id sh -c "pactl set-sink-mute @DEFAULT_SINK@ false; pactl set-sink-volume @DEFAULT_SINK@ +5%" ;;
esac
EOF

chmod +x /etc/acpi/handlers/audio.sh
```

Handle backlight buttons:
```sh
cat << 'EOF' > /etc/acpi/events/backlight
event=video/brightness.*
action=/etc/acpi/handlers/backlight.sh %e
EOF

cat << 'EOF' > /etc/acpi/handlers/backlight.sh
#!/bin/sh

backlight_instance=$(ls /sys/class/backlight | head -n 1)
backlight=/sys/class/backlight/$backlight_instance

step=$(expr $(< $backlight/max_brightness) / 20)

case $2 in
  BRTDN) echo $(($(< $backlight/brightness) - $step)) > $backlight/brightness;;
  BRTUP) echo $(($(< $backlight/brightness) + $step)) > $backlight/brightness;;
esac
EOF

chmod +x /etc/acpi/handlers/backlight.sh
```


## KeePassXC
```sh
pacman --quiet --sync --needed --noconfirm keepassxc
sed --in-place '/^xscreensaver/a keepassxc &' "/home/${USER}/.xinitrc"
```


## Change User Home Owner and Group

```sh
chown --recursive "${USER}:users" "/home/${USER}"
```
