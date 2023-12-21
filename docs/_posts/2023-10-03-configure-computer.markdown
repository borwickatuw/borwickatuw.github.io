---
layout: default
title: "Configuring a new MacBook"
---
I recently got a new computer, and I thought I'd try to track all the steps needed for me to set it up! Here's what I made a note of.

# Apple ID #

This requires access to the previous computer to get a confirmation code.

# Installing programs #

Installed:

  - Chrome
  - Dropbox, needed for `emacs.d` configuration
    - requires access to another device with the Dropbox account on it for authorization code
  - Emacs for Mac OS X
  - Google Drive for Desktop
  - LastPass
    - requires Apple ID
    - requires access to email to get LastPass approval
  - MacPorts
    - use saved list of requested ports if possible: `port echo requested | cut -d ' ' -f 1 | uniq > requested-ports.txt`
    - `cat requested-ports.txt | xargs sudo port install`
  - [Monosnap](https://monosnap.com)
  - Office: log into `office.com` &rarr; "Install apps"
  - OneTab for Safari
  - [Rectangle](https://rectangleapp.com/)
  - Vagrant
    - `vagrant-vmware-desktop` plugin
  - VMWare Fusion
  - XCode
    - `sudo xcodebuild -runFirstLaunch`
  - Zoom

# UI configuration #

  - System preferences
    - Map caps lock to control
    - Keyboard shortcuts > Input Sources: remove ctrl+space
    - Keyboard shortcuts > Function Keys: enable
    - Keyboard shortcuts > Modifier Keys: map caps lock to control
    - Spelling and prediction > disable "Correct spelling automatically" and "Capitalize words automatically"
  - Terminal
    - Profiles > Keyboard > "use Option as Meta key" for each profile
    - Create a 2x2 grid of terminal windows and save them as a "window group"
  - Fonts
    - Install [Julia Mono](https://juliamono.netlify.app)

# Shell configuration #

    ln -s ~/Library/CloudStorage/Dropbox ~/Dropbox
	mkdir ~/code
	ssh-keygen -t ed25519
	git clone ~/Dropbox/emacs.d ~/.emacs.d
    touch ~/.emacs.d/custom.el
	# add /opt/local/bin/bash to /etc/shells
	chsh

	# with bashmarks installed:
	#
	# `s` saves a bookmark for the current folder
	cd ~/code
	for i in *; do pushd $i; s $i; popd; done
	
# GitHub #

  - Add `~/.ssh/id_ed25519.pub` to GitHub keys + authorize for SSO

# Secrets #

  - Generate GPG key
  - Copy ~/.authinfo.gpg
  - Copy ~/build/M365-IMAP/config.py

# Email #

  - Comment out pre- and post- sync hooks in ~/.offlineimaprc
  - Run offlineimap -- this takes a long time
