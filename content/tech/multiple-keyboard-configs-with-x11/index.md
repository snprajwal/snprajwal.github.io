+++
date = 2023-03-27T00:46:23+05:30
title = "Managing multiple keyboard configurations with X11"
description = "Demystifying the horrors of xorg.conf"
tags = ['linux']
+++

Have you ever wanted to have multiple keyboards connected to the same device, each with a different configuration? Probably not. But in my case, I use Colemak as my standard keyboard layout. Normally, one would change their keyboard layout on the software level (in Linux, you can achieve this with `setxkbmap` for X11 and `localectl` for the TTY) and everything works. But I also use a handmade split keyboard with Colemak on the firmware. This means that I can type without changing the layout and it will still work like Colemak, since the firmware is handling the keycodes. If I do change the layout, it'll remap my Colemak input to Colemak a second a time and bungle everything. But this also means that my laptop keyboard is stuck with QWERTY. *Blasphemy O_O*

# X11 configs to the rescue

Luckily, X11 allows configuring inputs using the `setxkbmap` tool. For persistent configuration, we can create a file in `/etc/X11/xorg.conf.d` and pray to the benovelent Linux gods to make it work. Here are some important points about this special directory that the X server uses:

- The configuration language is... I have no clue. Good luck finding tutorials or guides for it. The only good documentation I managed to come across was [this](https://www.x.org/releases/current/doc/man/man5/xorg.conf.5.xhtml), and it's cryptic at best - one of the reasons why I'm writing this article. Hopefully I can save some poor soul the trauma of trying to understand how all this works.
- The X server processes the configs in the order of the numbers prefixed to the file name, e.g. `00-monitor.conf` is processed before `20-keyboard.conf`.
- The rules in each file are processed top to bottom, with the one at the very end of the file having highest precedence. Hence, the more general the constraints, the higher up the rule should be in the file.
- It's fiddly at best, and completely useless at worst. In most cases, it just silently fails, and the only debugging info you get is a bunch of `Error reading file` messages strewn across the log file. Very helpful =_=

Now let's get to actually writing the config. The basic idea is that you fetch the details of the input device with `xinput list --long`. This lists the product and vendor names of all input devices, with the two fields separated by... you guessed it - a goddamn whitespace. How do you know where the product name ends? X says that's a you problem, figure it out!

Once you manage to decipher what the name of your device is, it's just a matter of plugging that into the `MatchProduct` field in the config. If you don't specify this field, X will match all valid input devices that satisfy the other constraints in the section. Since I also use a Sanskrit layout occasionally, I just configured that with the `XkbLayout` and `XkbVariant` options. Note that I'm placing a comma before `san-kagapa` to indicate that I want to use the default variant for the US layout. The `XkbOptions` option lets us choose a handy shortcut to cycle through our layouts, and I went with **Alt+Space** since it was the most convenient for me. Here's what the final config looks like:

```bash
Section "InputClass"
        Identifier "keyboard catchall"
        MatchIsKeyboard "on"
        Option "XkbLayout" "us,in"
        Option "XkbVariant" "colemak,san-kagapa"
        Option "XkbModel" "pc105+inet"
        Option "XkbOptions" "grp:alt_space_toggle"
EndSection

Section "InputClass"
        Identifier "Brisingr Keyboard"
        MatchIsKeyboard "on"
        MatchProduct "Brisingr Keyboard"
        Option "XkbLayout" "us,in"
        Option "XkbVariant" ",san-kagapa"
        Option "XkbModel" "pc105+inet"
        Option "XkbOptions" "grp:alt_space_toggle"
EndSection
```

Bingo! The first section matches against every keyboard and sets the layout to Colemak. This takes care of my laptop keyboard. The second section sets the layout to QWERTY only for the keyboard whose product name matches "Brisingr Keyboard", which happens to be what I've named my split keyboard. No more double remapped garbled mess of letters!

If you're interested in seeing all the available config options for `setxkbmap`, surprise surprise - it's not avaliable in the man pages and requires considerable hunting to find on your local system. Here's a [GitHub Gist](https://gist.github.com/jatcwang/ae3b7019f219b8cdc6798329108c9aee) with all of them.
