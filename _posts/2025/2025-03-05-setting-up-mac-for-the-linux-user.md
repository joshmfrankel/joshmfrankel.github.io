---
layout: post
title: Setting up Mac for the Linux user
categories:
  - articles
tags:
  - mac
  - linux
  - setup
---

So you've got a new Mac. Maybe it's a work laptop, maybe you are switching from another operating system. Either way, if you're like me, you've got some set habits with your work environment. This article details all the configurations I do when setting up a new work laptop.

<!--excerpt-->

## Keyboard

The first (and arguably most important) step of this guide is adjusting your modifier key mappings. This stems from my biggest gripe with Mac OSX. The **Command (⌘)** key is used for copying, pasting, switching workspaces, and all other **Ctrl (^)** key related functionality. I've grown accustomed to how Linux sets its key mappings, which flips these two keys.

Luckily, Mac OSX has the concept of custom modifier keys, which define actions for several keys. Using spotlight search or **System Settings->Keyboard->Keyboard Shortcuts->Modifier Keys**, you can access the modifier keys configuration page.

![Spotlight Modifier Key](/img/2025/setting-up-mac-for-the-linux-user/spotlight-modifier-key.png "Mac Spotlight search for modifier keys")

Once on the page, you can switch the **Control (^)** and **Command (⌘)** keys to their opposite settings. A further step that I take is to map the **CapsLock** key to also perform **Command (⌘)** modifier actions. Because when is the last time you actually used your CapsLock key? Pretty much never for me.

![Modifier Keys Settings](/img/2025/setting-up-mac-for-the-linux-user/modifier-keys-settings.png "Modifier keys settings menu")

The great thing about changing the modifier keys, is that it applies everywhere system-wide. Previously, I've used tools like [Karabiner-elements](https://karabiner-elements.pqrs.org/) which give you granular control over application and system shortcuts, but that comes with lots of configuration and some idiosyncrasies.

### Spotlight

- Show spotlight search — `Control (^) + Space`

![Spotlight Shortcuts](/img/2025/setting-up-mac-for-the-linux-user/spotlight-shortcuts.png "Spotlight shortcut settings")

### Screenshots

- Copy Picture of selected area to clipboard — `F13 (Printscrn)`

![Screenshot Shortcuts](/img/2025/setting-up-mac-for-the-linux-user/screenshots.png "Screenshot shortcut settings")

### Mission Control

Set movement of workspaces to utilize the newly mapped `Control (^) key`
![Mission Control Shortcuts](/img/2025/setting-up-mac-for-the-linux-user/mission-control.png "Mission Control Shortcuts")

With that, you have basic keyboard configuration setup. Next up let's adjust the Mouse and Trackpad.

## Mouse and Trackpad

My preference is to disable Natural Scrolling for both mouse wheel and trackpad. This means that scrolling up on the mouse wheel moves the scrollbar up the page instead of down.

![Mouse settings](/img/2025/setting-up-mac-for-the-linux-user/mouse-settings.png "Mouse settings")
![Trackpad scroll settings](/img/2025/setting-up-mac-for-the-linux-user/trackpad-scrolling.png "Trackpad scroll settings")

Additionally, I like having the Trackpad respond from a single click to perform actions and the secondary click be set to the bottom right corner of the Trackpad. The single click to perform actions from my experience is on by default in several Linux distros.

![Trackpad pointer settings](/img/2025/setting-up-mac-for-the-linux-user/trackpad-pointer-settings.png "Trackpad pointer settings")

Next up let's update the Dock.

## Desktop & Dock

The Dock is my favorite part of both Mac OSX and many Linux distributions. If you're using Fedora Workstation or Elementary OS, the Dock in each of the these are fantastic. Mac's Dock is pretty good by default as well. I like to get information out of the way unless I need it, so hiding the dock until the mouse hovers over it is a key change I make.

![Dock Settings](/img/2025/setting-up-mac-for-the-linux-user/dock-settings.png "Dock settings")

To accomplish this, you'll need to enable "Automatically hide and show the Dock". This hides the Dock until the mouse moves over it. Now by default this animation is rather slow. I believe it is trying to mimic a pressure reveal. However, I want this to be smooth and snappy. Luckily, some enterprising individuals have figured out the internal settings used to calculate both the delay for showing as well as the animation timing. Here is the result of making the change.

![Dock Animation](/img/2025/setting-up-mac-for-the-linux-user/dock-animation.gif "Dock Animation")

In order to configure this, you'll need to open your Terminal and run the following lines. `autohide-delay` is how long to wait before starting the show/hide transition, while `autohide-time-modifier` is the animation timing. You can see that we set it to zero in order for it to immediately start transitioning states.

```
defaults write com.apple.dock autohide-delay -int 0
defaults write com.apple.dock autohide-time-modifier -float 0.4
killall Dock
```

[Original source](https://apple.stackexchange.com/a/70598/612153)

Here are some additional Dock settings that I utilize for a better overall experience:

- Minimize windows using **Scale Effect** gives a more consistent animation transition experience.
- Enabling **Minimize windows into application icon** ensures that each application in the Dock provides all the open windows of itself. Otherwise, these open next to the Downloads and Trash Can for each window.

![Minimize app into Dock](/img/2025/setting-up-mac-for-the-linux-user/minimize-app-into-dock.png "Minimize app into Dock")

- Disabling "Show suggested and recent apps in Dock" helps to remove clutter in the Dock for a feature I never use.

### App set to specific workspace

I like to have 4 Workspaces when programming: Browser, Editor, Git Gui, and Spotify. So after opening an application you can set the application to always open in a specific Workspace with the context menu in the Dock.

![App set to specific workspace](/img/2025/setting-up-mac-for-the-linux-user/app-set-to-specific-workspace.png "App set to specific workspace")

### Control Center

Within here there are a couple of nice tweaks we can make in the **Menu Bar Only** section.

- Spotlight set to "Don't show in Menu Bar". Spotlight is more useful from a keyboard shortcut.
- Automatically hide and show the menu bar set to "Always". This will hide your top menu bar and allow more focus on current work (as well as more screen real estate).
- Recent Documents, applications, and servers set to "None". I prefer to have the quick open functionality focus only on what is currently open. So right clicking on a Dock icon will show me all the windows I have currently open without past open files.

![Recent documents settings](/img/2025/setting-up-mac-for-the-linux-user/recent-documents.png "Recent documents settings")

### Mission Control

Now on the same settings page under "Mission Control", the first option "Automatically rearrange spaces based on use" should be disabled. This prevents your workspaces from changing their order based on usage. I have my workspaces setup in the following order: Browser & Slack, Editor, Git Merge Tool, and Spotify. This allows me a consistent development experience to easily move back and forth between tasks.

Much like the Dock's show/hide timing, we can adjust Workspace switching by setting the following value:

```
defaults write com.apple.dock workspaces-swoosh-animation-off -bool YES
killall Dock
```

This can help, but there is still some slowness from switching spaces, depending on when you activate the keyboard shortcut to switch. I'm not aware of other methods to speed switching up further.

Next up are several general System Settings.

## Appearance

Personally I prefer a dark theme as it is easier on the eyes. I do enjoy a good pop of color, so I select the Purple accent for the system.

![Apperance Settings](/img/2025/setting-up-mac-for-the-linux-user/appearance-settings.png "Apperance Settings")

## Display

I use a laptop connected to a primary monitor. I've used a single monitor for years and years, so I don't use the laptop for anything but powering the screen. Because of this, I've set my display mode to Mirror screen.

## Homebrew

The defacto package manager for Mac. Get it as it is nearly guaranteed that at some point you'll need it.

[https://brew.sh/](https://brew.sh/)

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Applications

Over the years, I've come to use the following applications consistently. I won't go into too much more detail around application configuration until a follow-up post, but here is the list of apps that I use. Coming from a Linux background, I have focused on apps which are free. The only fully paid app below is Magnet (window snapping) while Sublime Text, Sublime Merge, and Cursor have free versions with less functionality.

- [1Clipboard](https://1clipboard.io/) - For Clipboard management
- [Brave Browser](https://brave.com/) - For Ad-free web browsing
- [Docker](https://www.docker.com/) - For Containerization
- [Giphy Capture](https://apps.apple.com/us/app/giphy-capture-the-gif-maker/id668208984?mt=12) - For recording short animated demos in Gif format
- [Hidden Bar](https://apps.apple.com/us/app/hidden-bar/id1452453066?mt=12) - For customizing the top bar application tray
- [iA Writer](https://ia.net/writer) - For focused writing
- [Bruno](https://www.usebruno.com/) - For API development
- [Magnet](https://apps.apple.com/us/app/magnet/id441258766?mt=12) - For enhanced application tiling
- [MeetingBar](https://meetingbar.app/) - For meeting notifications and joining video calls
- [Spotify](https://www.spotify.com/us/download/mac/) - For vibes
- [Slack](https://slack.com/) - For chatting
- [Sublime Merge](https://www.sublimemerge.com/) - For Git Client
- [Sublime Text](https://www.sublimetext.com/) and [Cursor](https://www.cursor.com/) - For Code editing
- [Tabby](https://tabby.sh/) - For modern terminal

### NoTunes

This is a nice quality of life change for how the media buttons interact with the default Mac audio applications.
Specifically, it can be used to change the default program to something like Spotify instead of Music or Tunes.

[NoTunes](https://github.com/tombonez/noTunes)

1. Install - `brew install --cask notunes`
2. Right click or control-click the menu bar icon and click Hide Icon.
3. Replace with Spotify or other app - `defaults write digital.twisted.noTunes replacement /Applications/Spotify.app`

## Conclusion

Now we've ironed out a bunch of rough edges for those of us coming from a Linux environment. Such smooth. Many nice. Stay tuned for a similiar post regarding my Fedora Workstation setup.

Got any Mac OSX tweaks or settings you use? Leave a comment below to get the discussion started.
