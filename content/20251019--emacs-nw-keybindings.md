+++
title = "Emacs in the terminal: keybindings, client and more"
slug = "emacs-in-terminal-ergonomics"
date = 2025-10-19
tags = ["emacs"]
categories = ["guide"]
+++

# Emacs in the terminal

If you've ever tried using Emacs with `emacs -nw` (no window) in your terminal,
you might've faced a frustrating experience: because Emacs binds keys starting
with <kbd>Ctrl</kbd> and terminal input is first handled by the terminal
emulator; some keys might not work as expected. They might provoke an action in
the terminal but not in Emacs or be changed on the way. If you're also not using
a "typical" QWERTY layout or have the bad idea of having accents (thus non ASCII
characters) in your language, the problem gets worse.

A common issue is trying to use <kbd>Ctrl+Backspace</kbd> which should delete a
word but instead inserts <kbd>Ctrl+h</kbd> instead when used in the terminal.
There are also issues for more complex key/characters combinations.

## Fix

The fix consists of using the [Kitty Keyboard
Protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) in both Emacs and
your terminal emulators.

I've tried to set a couple variables and tried some terminal emulators; the best
experience I've had is with the [`kkp` package](https://github.com/benotn/kkp)
and the [ghostty terminal emulator](https://ghostty.org/).

With the following configuration, in your initialization file:

```elisp
(use-package kkp
  :ensure t
  :config
  ;; (setq kkp-alt-modifier 'alt) ;; use this if you want to map the Alt keyboard modifier to Alt in Emacs (and not to Meta)
  (global-kkp-mode +1))
```

## Quality of life

When setting Emacs as your `$EDITOR` in the shell, I discovered that `export
EDITOR="emacs -nw"` sometimes fail depending on your shell, because it expects
an executable without any white-spaces. An alternative is to write a shell
function and refer to that as the editor:

```sh
#!/bin/sh
exec emacs -nw "$@"
```

I have it in `/home/natfu/.local/bin/emacseditor` and I've set my `$EDITOR` to this path.

Note that this spins a new instance of Emacs every time with a slight delay in starting
up even with a rather optimized `~/.emacs.d/init.el`. The alternative is to use the
[`emacsclient`](https://www.gnu.org/software/emacs/manual/html_node/emacs/Invoking-emacsclient.html).
In my case, I'm using Linux with `systemd` so I can write a unit file that will start an
Emacs daemon in the background and open a new frame instantly by calling it. On my
system, I've found that file in those places:

- `/usr/share/emacs/30.2/etc/emacs.service`
- `/usr/lib/systemd/user/emacs.service`

To make it clear that I'll use it for my user, I'll copy the file under
`~/.config/systemd/user/emacsd.service`. You can tweak it to your liking, for
example by adding an environment file only for that service (useful to set the
path among other things).

```systemd
[Unit]
Description=Emacs text editor (GUI)
Documentation=info:emacs man:emacs(1) https://gnu.org/software/emacs/

[Service]
Type=notify
ExecStart=/usr/bin/emacs --fg-daemon
ExecStop=/usr/bin/emacsclient --eval "(kill-emacs)"

# Emacs will exit with status 15 after having received SIGTERM, which
# is the default "KillSignal" value systemd uses to stop services.
SuccessExitStatus=15

# The location of the SSH auth socket varies by distribution, and some
# set it from PAM, so don't override by default.
# Environment=SSH_AUTH_SOCK=%t/keyring/ssh
Restart=on-failure

[Install]
WantedBy=default.target
```

Then, I can change the `/home/natfu/.local/bin/emacseditor` file to use the client
instead.

```sh
#!/bin/sh
exec emacsclient -t -a="" "$@"
```

Finally, start the service with `systemctl --user start --now emacsd.service` and
`systemctl --user enable --now emacsd.service` to have the service launch Emacs for you
at the start of a user session.

As always, the [Arch wiki on Emacs](https://wiki.archlinux.org/title/Emacs) is great
information, even if you're not using Arch.

### Important note for Linux

On most Linux systems, you'll be able to use [`.desktop`
files](https://wiki.archlinux.org/title/Desktop_entries) which not only do let
you add icons and interact on a GUI level with an application (like icons in
menus/desktops), it also lets you interact with the rest of the system, for
example through shortcuts.

From my understanding, it's up to the distribution and package management to
create those files, here's what Arch is doing for Emacs. Where the entry called
'Emacs (Client)' when clicked on, will try to create a frame by connecting to
the client if it exists and create it otherwise. It lives in
`/usr/local/share/applications/emacsclient.desktop`. If you want to create one
that'll be picked up by system instead of that one, you should use
`~/.local/share/applications/emacsclient.destkop`.

```systemd
[Desktop Entry]
Name=Emacs (Client)
GenericName=Text Editor
Comment=Edit text
MimeType=text/english;text/plain;text/x-makefile;text/x-c++hdr;text/x-c++src;text/x-chdr;text/x-csrc;text/x-java;text/x-moc;text/x-pascal;text/x-tcl;text/x-tex;application/x-shellscript;text/x-c;text/x-c++;x-scheme-handler/org-protocol;
Exec=sh -c "if [ -n \\"\\$*\\" ]; then exec /usr/local/bin/emacsclient --alternate-editor= --reuse-frame \\"\\$@\\"; else exec emacsclient --alternate-editor= --create-frame; fi" sh %F
Icon=emacs
Type=Application
Terminal=false
Categories=Development;TextEditor;
StartupNotify=true
StartupWMClass=Emacs
Keywords=emacsclient;
Actions=new-window;new-instance;

[Desktop Action new-window]
Name=New Window
Exec=/usr/local/bin/emacsclient --alternate-editor= --create-frame %F

[Desktop Action new-instance]
Name=New Instance
Exec=emacs %F
```

Right now, I'm keeping the systemd services and just let that file pick up the
existing client.

I've also bound calling the client to <kbd>Super+E</kbd> to get a new frame
quickly wherever I am on the system and aliased `e=$EDITOR` to open the editor
in the terminal.

## Using multiple Emacs clients

I've had a couple annoyances since the Emacs configuration is loaded once when
the client is created and changes, like themes, are reflected on the client and
all frames connecting to it.

I generally change my theme based on the time of day in the GUI but I want it to
stay a dark theme in the CLI in general. I also want to load some packages only
in graphical mode, like `org-mode`. For example,

```elisp
(use-package org
  :pin gnu
  :ensure nil
  :defer t
  :diminish "Οrg"
  :if (display-graphic-p)
  :custom
  ;; ... custom
  :config
  ;; ... config)
```

This prompted me to write a systemd service file for the GUI and the other for
the TUI. Which Emacs lets us do easily: you can name the socket you expose the
client on.

> This is getting out of hand, now there are two of them!

```systemd
[Unit]
Description=Emacs text editor (GUI)
Documentation=info:emacs man:emacs(1) https://gnu.org/software/emacs/

[Service]
Type=notify
ExecStart=/usr/bin/emacs --fg-daemon=gui
ExecStop=/usr/bin/emacsclient -s gui --eval "(kill-emacs)"

# Emacs will exit with status 15 after having received SIGTERM, which
# is the default "KillSignal" value systemd uses to stop services.
SuccessExitStatus=15

# The location of the SSH auth socket varies by distribution, and some
# set it from PAM, so don't override by default.
# Environment=SSH_AUTH_SOCK=%t/keyring/ssh
Restart=on-failure

[Install]
WantedBy=default.target
```

and

```systemd
[Unit]
Description=Emacs text editor (TUI)
Documentation=info:emacs man:emacs(1) https://gnu.org/software/emacs/

[Service]
Type=notify
ExecStart=/usr/bin/emacs --fg-daemon=tui
ExecStop=/usr/bin/emacsclient -s tui --eval "(kill-emacs)"

# Emacs will exit with status 15 after having received SIGTERM, which
# is the default "KillSignal" value systemd uses to stop services.
SuccessExitStatus=15

# The location of the SSH auth socket varies by distribution, and some
# set it from PAM, so don't override by default.
# Environment=SSH_AUTH_SOCK=%t/keyring/ssh
Restart=on-failure

[Install]
WantedBy=default.target
```

As a result, the desktop file needs to refer to the `gui` client and the editor
in the terminal needs to refer to the `tui` client.

```systemd
[Desktop Entry]
Name=Emacs (Client)
GenericName=Text Editor
Comment=Edit text
MimeType=text/english;text/plain;text/x-makefile;text/x-c++hdr;text/x-c++src;text/x-chdr;text/x-csrc;text/x-java;text/x-moc;text/x-pascal;text/x-tcl;text/x-tex;application/x-shellscript;text/x-c;text/x-c++;x-scheme-handler/org-protocol;
Exec=sh -c "if [ -n \\"\\$*\\" ]; then exec /usr/local/bin/emacsclient -s gui --reuse-frame \\"\\$@\\"; else exec /usr/local/bin/emacsclient -s gui --create-frame; fi" sh %F
Icon=emacs
Type=Application
Terminal=false
Categories=Development;TextEditor;
StartupNotify=true
StartupWMClass=Emacs
Keywords=emacsclient;
Actions=new-window;new-instance;

[Desktop Action new-window]
Name=New Window
Exec=/usr/local/bin/emacsclient -s gui --create-frame %F

[Desktop Action new-instance]
Name=New Instance
# Use background daemon that will die when the frame is closed
Exec=emacs --daemon=gui-temp -Q && emacsclient -s gui-temp --create-frame %F
```

```sh
#!/bin/sh
exec emacsclient -s tui -t -a="" "$@"
```

Finally, you can reload the systemd daemon and start then enable the services:

```sh
systemctl --user daemon-reload
systemctl --user enable emacsd-gui.service emacsd-tui.service
systemctl --user start emacsd-gui.service emacsd-tui.service
```

That's all for me, I honestly didn't think that trying to fix
<kbd>Ctrl+Backspace</kbd> would become such an adventure.
