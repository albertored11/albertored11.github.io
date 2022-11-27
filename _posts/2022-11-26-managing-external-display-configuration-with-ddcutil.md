---
title: Managing external display configuration with ddcutil
date: 2022-11-27 14:47:00 +0200
categories: [Linux, Power management]
tags: [linux, bash, script, keyboard, display, ddcutil, sxhkd]
---

Almost every laptop keyboard include two function keys: one to decrease the
brightness of the screen and other to increase it.

In most Linux desktop environments, those two keys work out-of-the-box for the
laptop display, but... what about external displays?

Recently I got a new keyboard, and I was surprised to see those brightness keys.
Since I'm using it for my desktop PC, at first I thought they would be useless
and I assigned them to other actions.

However, a few days ago, I learnt about the existence of
[DDC/CI](https://en.wikipedia.org/wiki/Display_Data_Channel#DDC.2FCI), a Common
Interface standard that allows the computer to send commands to the display and
to receive data from it over [I2C](https://wiki.archlinux.org/title/I2C), a
serial communication bus.
Those communication can be done in Linux thanks to the **i2c-dev** kernel
module and, with a command line utility like
[ddcutil](https://www.ddcutil.com/), it is as easy as issuing some commands in
our shell.

We're going to learn how to install and configure ddcutil and to use it to
control the brightness of an indefinite number of external displays and to
turn them off easily.

Of course, displays must be compatible with the aforementioned standards. But
don't worry, most modern ones are!

## Installing and configuring ddcutil

### Installation

ddcutil is available as a package in most distributions.

For example, for Arch Linux:

```console
$ sudo pacman -S ddcutil
```

### Configuration

Then,
[some additional configuration steps are necessary](https://www.ddcutil.com/config_steps/).

#### Load i2c-dev module

**i2c-dev** kernel module is required by ddcutil.

Check if it is already available (e.g. if it is built into the kernel):

```console
$ lsmod | grep -F i2c_dev
```

If not, it must be loaded explicitly:

```console
$ sudo bash -c 'echo i2c-dev > i2c-dev.conf'
```

A reboot is needed afterwards.

#### Give permissions to I2C devices

On some distributions, including Debian, Ubuntu and Arch Linux, all I2C devices
are assigned to the group **i2c**. This is done by the **i2c-tools** package,
which is a dependency of ddcutil.

In order for your user to have access to the devices, it must be added to that
group:

```console
$ sudo usermod $USER -aG i2c
```

## Controlling brightness

### Detect displays

First of all, run the following command to detect available displays:

```console
$ ddcutil detect
```

You will get an output like this:

```
Display 1
   I2C bus:  /dev/i2c-1
   DRM connector:           card0-HDMI-A-2
(ommited output)

Display 2
   I2C bus:  /dev/i2c-4
   DRM connector:           card0-DP-1
(ommited output)
```

With that output, you can run commands that apply to the first display using the
option `--display 1` or `--bus 1` (because it uses the bus `/dev/i2c-1`), or to
the second display with `--display 2` or `--bus 4`.

### Get/set brightness

Displays have some settings coded into what is called VCP features. Their names
and their values can be queried using the `getvcp` command.

Take a look at all of them for one display:

```console
$ ddcutil getvcp ALL --bus 1
```

You will get the code and the value for each one. This is the one for
brightness:

```
VCP code 0x10 (Brightness                    ): current value =   100, max value =   100
```

That means brightness feature has code `10`, its current value is `100` and its
maximum value is `100`.

That value can be set using the `setvcp` command:

```console
$ ddcutil setvcp 10 50 --bus 1
```

That command would set brightness to 50 % for display with bus number 1.

It also works with deltas:

```console
$ ddcutil setvcp 10 + 5 --bus 1
$ ddcutil setvcp 10 - 5 --bus 1
```

Those commands would respectively increase and decrease brightness by 5 %.

#### Set brightness in all displays at once

ddcutil doesn't have an option to run a command for multiple displays. A
workaround is using xargs. Assuming you have 2 displays with bus numbers 1 and
4 and you want to increase brightness by 5 % for both of them:

```console
$ echo "1 4" | xargs -n1 -P2 sh -c 'ddcutil setvcp 10 + 5 --bus $1' sh
```

would do the trick.

**Important:** use bus numbers rather than display numbers to reference
displays. Otherwise, parallel commands won't work.

## Turning displays off

Another interesting VCP feature is **Power mode**, which can be set to turn on
and off the display. In my case, it only works to turn them off, but I guess it
depends on the display model.

The VCP code for this feature is **d6**:

```console
$ ddcutil getvcp d6 --bus 1
```

```
VCP code 0xd6 (Power mode                    ): DPM: On,  DPMS: Off (sl=0x01)
```

In order to turn off the display, the value has to be set to **5** and, to turn
it on, it has to be set to **1**, so

```console
$ ddcutil setvcp d6 5 --bus 1
```
would turn off display with bus number 1.

## Control brightness using the keyboard

Let's get to the interesting part: doing all this conveniently using the
keyboard!

### Shell script

The following script can be used to easily set brightness in all displays to a
fixed value or to increase or decrease it, or to turn them off.

```bash
#!/usr/bin/env bash

progname="$(basename "$0")"

usage() {
    echo "usage: $progname <up/down/set/off> [brightness %]" >&2
    exit 1
}

[[ "$#" -eq 1 || "$#" -eq 2 ]] || usage

# Exit if an instance of the script is already running to avoid errors
# It has to be somewhere in $PATH for this to work
n_px=$(pgrep -f "^bash $(which $progname)" | wc -l)
[[ $n_px -gt 2 ]] && exit 1

# Get bus number for all displays and number of displays
displays=$(ddcutil detect --terse | grep -F "I2C bus:" | awk -F '-' '{print $2}')
n_displays=$(echo $displays | wc -w)

case "$1" in
    up) 
        b="+\ $2"
    ;;  
    down)
        b="-\ $2"
    ;;  
    set)
        b="$2"
    ;;
    off)
        # Turn off all displays
        echo $displays | xargs -n1 -P$n_displays sh -c 'ddcutil setvcp d6 5 --bus $1' sh
        exit 0
    ;;
    *)
        usage
    ;;
esac

conf=$(for m in $displays; do echo $m $b; done)

echo $conf | xargs -n2 -P$n_displays sh -c 'ddcutil setvcp 10 $2 --bus $1' sh
```

You can find this script
[in my dotfiles](https://github.com/albertored11/dotfiles/blob/pc-nord/.local/bin/bright-notif),
which includes an extra feature to use [xob](https://github.com/florentc/xob).

Save this script somewhere in your `$PATH`. The following commands can be used,
assuming we named it `brightctl`:

* `up`/`down`: increase/decrease brightness by x % (e.g. `brightctl up 5`).
* `set`: set brightness to x % (e.g. `brightctl set 80`).
* `off`: turn displays off (e.g. `brightctl off`).

### Keybindings

Now, we can assign calls to the script to the keyboard using, for example,
sxhkd.

You can add this to your `sxhkdrc`:

```
# brightness up/down
XF86MonBrightness{Up,Down}
    bright-notif {up,down} 5

# custom brightness
ctrl + XF86MonBrightness{Up,Down}
    bright-notif set {100,30}
```

This way, brightness can be increased/decreased by 5 % using the brightness
keys and, with the Control modifier, it can be set to 30 % or 100 %.

You will notice that running one of these commands is much slower than when
controlling the brightness of a native display.

## Turning displays off conveniently

### Shell script

We can use the turning off displays feature at our convenience: for example,
after before locking the screen or just before suspending the system.

I use a script to lock the screen with two options: turning off displays and
suspending the system.

```bash
#!/usr/bin/env bash

usage() {

printf "%s" "\
usage: $(basename "$0") [options]

Options:
    -h, --help                  show this help message and exit.
    -o, --off                   turn all displays off.
    -s, --suspend               run systemctl suspend afterwards.

"

exit 1

}

get_args() {
    TEMP=$(getopt \
               -l 'help,off,suspend' \
               -o 'hos' \
               -n "$(basename "$0")" \
               -- "$@") || exit 1

    eval set -- "$TEMP"
    unset TEMP

    while true; do
        case "$1" in
            '-h' | '--help')
                usage
            ;;
            '-o' | '--off')
                off=1
                shift
                continue
            ;;
            '-s' | '--suspend')
                susp=1
                shift
                continue
            ;;
            '--')
                shift
                break
            ;;
            *)
                printf "error\n" >&2
                exit 1
            ;;
        esac
    done
}

main() {
    get_args "$@"

    [[ -z "$(pgrep -f ^i3lock)" ]] || exit 1

    # Comment if not using dunst
    killall -SIGUSR1 dunst

    [[ "$off" ]] && bright-notif off

    # Use your favorite i3lock options
    i3lock [options] &

    if [[ "$susp" ]]; then
        sleep 0.5
        systemctl suspend
    fi

    wait

    # Comment if not using dunst
    killall -SIGUSR2 dunst

    return 0
}

main "$@"
```

You can also find this script
[in my dotfiles](https://github.com/albertored11/dotfiles/blob/pc-nord/.local/bin/lock-screen).

When using `--off` option, the displays are turned off before locking the
screen.  When using `--suspend` option, the system is suspended after locking
the screen.

### Keybindings

You can add this to your `sxhkdrc`:

```
# lock screen
super + {_,alt + }l
    lock-screen{_, --off}

# suspend
super + shift + s 
    lock-screen --off --suspend
```

So, this way

* with Super + L, the screen is locked
* with Super + Alt + L, the displays turn off and the screen is locked
* with Super + Shift + S, the displays turn off, the screen is locked and the
system is suspended

{% include giscus.html %}
