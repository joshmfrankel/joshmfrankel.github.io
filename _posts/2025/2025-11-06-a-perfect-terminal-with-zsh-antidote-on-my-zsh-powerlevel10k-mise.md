---
layout: post
title: A Perfect terminal with Zsh, Antidote, Oh My Zsh, Powerlevel10k, and Mise.
categories:
  - tutorials
tags:
  - zsh
  - terminal
  - linux
  - mac osx
  - version managers
---

I love to customize my development environment. Between operating system, editor, and terminal, I'm always reading through the configuration options to improve my workflow. Getting it to look pretty is also great since
I spend so much time working with these tools. This article details my current setup for crafting a perfect terminal with Zsh, Antidote, Oh My Zsh, Powerlevel10k, and Mise.

<!--excerpt-->

## ZSH - An Enhanced Shell

![Zsh](/img/2025/perfect-terminal/zsh.png "Image source https://www.zsh.org")

First off, we'll want to configure the base shell that our terminal will utilize. A lot of plugin ecosystems have strong integrations with Zsh so we'll use that.

If you're on Mac, congrats! Zsh is installed by default.

For Linux, you'll need to [install it, depending upon your distribution](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH#ubuntu-debian--derivatives-windows-10-wsl--native-linux-kernel-with-windows-10-build-1903). Here are two popular distribution installation commands:

- Fedora - `dnf install zsh`
- Debian - `apt install zsh`

Once installed, configure your terminal to use ZSH as its shell with: `chsh -s /bin/zsh`. You can double check this is correctly setup by typing `echo $SHELL` to print the current shell environment.

![Zsh shell example](/img/2025/perfect-terminal/zsh-shell.png)

Alright, we've got the foundation set for adding additional functionality in the form of plugins.

## Antidote - A Zsh plugin manager

![Antidote](/img/2025/perfect-terminal/antidote.png "Image source https://antidote.sh")

[Antidote](https://antidote.sh/) is a Zsh plugin manager designed to be efficient and minimal. It is the latest evolution of the **Antigen -> Antibody -> Antidote** ecosystem. Antigen was the original, but is no longer under active development. Antibody is written in Go and has been deprecated. Antidote is a return to form, being written in Zsh.

So first things first, let's install it.

```zsh
git clone --depth=1 https://github.com/mattmc3/antidote.git ${ZDOTDIR:-~}/.antidote
```

You'll now have an **.antidote** folder within your Home directory. This is where Antidote functionality will live. We need to inform our shell where this is located. Add the following to the **TOP** of your **.zshrc** file. It is generally recommended to keep this at the top or near the top of your shell to ensure it loads plugins first and efficiently.

```zsh
# source antidote
source ~/.antidote/antidote.zsh

# initialize plugins statically with ${ZDOTDIR:-~}/.zsh_plugins.txt
antidote load
```

Antidote also utilizes a plugins file in order to determine which plugins and libraries to load. To prepare for the next section, let's go ahead and create our plugins file:

```zsh
touch ~/.zsh_plugins.txt
```

Before we move on try running `antidote --version` to ensure it is installed correctly.

```zsh
❯ antidote --version
antidote version 1.9.7 (9be3256)
```

Time to add some fancy plugins.

## Zsh Plugins

![Zsh plugins](/img/2025/perfect-terminal/zsh-plugins.png "Image source https://github.com/zsh-users/")

First up, we'll introduce several core Zsh plugins. Autosuggest, syntax highlighting, and completions. Our **.zsh_plugins.txt** file will accept references to the related repository for the plugin, so the following is all that is needed within the file.

```zsh
# .zsh_plugins.txt
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting
zsh-users/zsh-completions
```

After saving this, if you reload your terminal with `source ~/.zshrc` you'll see Antidote load the new plugins. This is courtesy of the `antidote load` line. You can test these are working by testing the following:

- Typing an invalid command should highlight in red
- Typing a valid command should highlight in green
- Type a command. Try starting to type it again, and you should see a grayed out suggest for you to choose.

![Zsh autocorrect plugin example](/img/2025/perfect-terminal/zsh-autocorrect.png)

Something I like to add at this point are a couple useful aliases. These can be added to your **.zshrc** file inline or sourced from a separate file. Personally I like to have these in a separate directory **~/.config/zsh/**.

```zsh
# source antidote
source /path/to/antidote/antidote.zsh

# initialize plugins statically with ${ZDOTDIR:-~}/.zsh_plugins.txt
antidote load

##################
# Configurations #
##################

source ~/.config/zsh/zshrc_aliases
```

And to ensure the file exists run `touch ~/.config/zsh/zshrc_aliases` and add the following:

```zsh
# Aliases
alias reload="exec zsh"
alias config="vim ~/.zshrc"
alias plugins="vim ~/.zsh_plugins.txt"
```

Resource your terminal again, and now you can simply type `reload` to source the terminal. `config` and `plugins` allow for easy editing of the **.zshrc** and **.zsh_plugins.txt** files, respectively.

## Oh My Zsh - A Zsh framework

![Oh My Zsh](/img/2025/perfect-terminal/ohmyzsh-logo.png "Image source https://ohmyz.sh")

The most important benefit of using Antidote is being able to pick and choose which plugins you want to install. [Oh My Zsh](https://ohmyz.sh/) is a popular framework with a ton of functionality and plugins. However, many of the plugins you'll never end up using. I much prefer having a simple, minimal configuration, which is where Antidote shines.

> Oh My Zsh is a delightful, open source, community-driven framework for managing your Zsh configuration.
>
> [Oh My Zsh](https://ohmyz.sh)

Now, in order to cherry-pick specific plugins from other frameworks (Antidote can also grab plugins from Pretzo), you need to also include any related dependencies that those plugins require. This can add complexity, which we want to avoid. Thankfully, there is a recommended plugin which handles these dependencies called **use-omz**. Let's start by adding that to our **.zsh_plugins.txt** file.

```zsh
# .zsh_plugins.txt
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting
zsh-users/zsh-completions

# Oh-my-zsh dependency management
getantidote/use-omz
```

This will ensure we don't have to install separate dependencies for Oh My Zsh plugins. Now I really like several Oh My Zsh plugins such as: rails, git, and bundler along with their common library functions, but feel free to pick your own from the [plugins page](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins).

```zsh
# .zsh_plugins.txt
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting
zsh-users/zsh-completions

# Oh-my-zsh dependency management
getantidote/use-omz

# Make Zsh more featureful
# Git status is faster
# History and other enhancements
# See: https://github.com/ohmyzsh/ohmyzsh/tree/master/lib
ohmyzsh/ohmyzsh path:lib

# Oh-my-zsh plugins
ohmyzsh/ohmyzsh path:plugins/rails
ohmyzsh/ohmyzsh path:plugins/git
ohmyzsh/ohmyzsh path:plugins/bundler
```

Run `reload` to resource your terminal and watch the new plugins become installed.

You might be thinking, "Why didn't we install any themes?". We'll superpower our terminal with powerlevel10k which has an amazing theme including additional functionality.

## Powerlevel10k - A theme for Zsh

![Vegeta over 9000!](/img/2025/perfect-terminal/over-9000.gif "Image source https://tenor.com")

At first glance, Powerlevel10k is just a theme for Zsh. However, boiling it down to a simple theme doesn't give it proper credit. It is so much more with several prominent features including:

- No prompt lag - Entering command immediately loads next prompt ([documentation](https://github.com/romkatv/powerlevel10k?tab=readme-ov-file#uncompromising-performance))
- Instant prompt - First prompt uses minimal ([benchmarks](https://github.com/romkatv/zsh-bench#instant-prompt))
- Prompt Segments - Notably the git **vcs** segment is incredibly useful

![Powerlevel10k example](/img/2025/perfect-terminal/powerlevel10k-git.png)

- Transient Prompt - Trims prompt extras to make previous commands easier to copy-paste and read.

![Powerlevel10k transient prompt example](/img/2025/perfect-terminal/powerlevel10k-transient-prompt.png)

- Font Glyphs - Technically this is separate from Powerlevel10k but is highly recommended in the installation instructions.

Convinced? If so, let's install it.

Start off by adding to our plugins file for Antidote.

```zsh
# .zsh_plugins.txt
romkatv/powerlevel10k
```

Install the recommended font to enable glyphs. You'll need to download each of the four listed fonts and install them for your OS. Make sure to set your terminal application to utilize the newly installed font by setting "MesloLGS NF" as the default. If you're unsure how to do this with your terminal, [refer to the multitude of guides](https://github.com/romkatv/powerlevel10k?tab=readme-ov-file#manual-font-installation) on the Powerlevel10k repository.

Now, reload your shell source with our `reload` alias.

Powerlevel10k comes prebaked with its own configuration utility. Run `p10k configure` to kick off the process. This will walk you through the different configuration options. Here are the ones I use:

- Prompt Style - Rainbow
- Character Set - Unicode
- Prompt Separator - Angled
- Prompt Head - Angled
- Prompt Height - One Line
- Prompt Spacing - Sparse
- Icons - Many Icons
- Prompt Flow - Concise
- Transient Prompt - Yes
- Instant Prompt Mode - Verbose

One manual change, I always do with Powerlevel10k, is to remove the truncation logic around git branch name. By default, the git branch displayed in the prompt segment gets truncated down to 32 characters. I often need to know the entire branch name and at times copy it, so removing this limitation is important.

We can adjust this by editing the underlying **.p10k.zsh** file. Run `vim ~/.p10k.zsh` and look for the following lines:

```zsh
# Tip: To always show local branch name in full without truncation, delete the next line.
(( $#branch > 32 )) && branch[13,-13]="…"  # <-- this line
```

Following along with the suggestion, remove or comment the line out in order to return the fully qualified git branch name.

Now we have a supercharged terminal. Let's take it one more step to use the excellent **asdf** version manager for dependency management.

## One Version Manager to rule them all

Managing each tool's version can be tiresome. Having a version manager specific to each tool's version can be annoying. Having a single version manager to manage all your tool versions is excellent!

Mise and Asdf work as a version management framework for every programming language. Gone are the days of having to manage npm, rbenv, rvm, etc where as these will handle all languages. We can install the binary files and add them to our path to start adding programming languages.

> **Note**: I've layed out both options **mise** and **asdf**. Until recently I was only using **asdf** but I've found **mise** to be superior in terms of UX and ease-of-use.

### Mise - The front-end to your dev env

![Mise VM](/img/2025/perfect-terminal/mise.svg)

For Mac, you can use Homebrew to install Mise

```zsh
brew install mise
```

If you are using Linux, the following will install and activate **Mise** for ZSH.

```zsh
curl https://mise.run/zsh | sh
```

Mise actually has many different [installation guides](https://mise.jdx.dev/installing-mise.html#installing-mise) available which is quite nice for the various architectures.

Once installed, you can use the `mise use ruby@3.4.2` style command to install both the Ruby plugin as well as the specific version. Mise makes this operation happen at the same time unlike **asdf** which you'll see shortly.

> The .tool-versions file is asdf's config file and it can be used in mise just like mise.toml. It isn't as flexible so it's recommended to use mise.toml instead.
>
> - [Mise .tool-versions documentation](https://mise.jdx.dev/configuration.html#tool-versions)

Alternatively, you can craft a **mise.toml** file which is much akin to ASDF's **.tool-versions** file, and run `mise install` within the same directory to install all specified tools. Here's my mise.toml file in my home directory:

```zsh
[tools]
ruby = '3.4.2'
python = '3.13.2'
node = '22.13.1'
```

Helpfully, Mise has great UX with commands that are intuitive and just make sense. If you ever need to check which version of a tool you are running and from which directory it is being set from you can run the `mise ls` command to return all available versions. Here's an example:

```zsh
❯ mise ls
Tool    Version  Source        Requested
node    20.16.0
node    22.13.1  ~/.mise.toml  22.13.1
node    22.15.0
python  3.13.2   ~/.mise.toml  3.13.2
ruby    3.4.2    ~/.mise.toml  3.4.2
```

There are many other helpful commands that Mise provides which can be [found here](https://mise.jdx.dev/walkthrough.html#common-commands).

### Asdf - The Multiple Runtime Version Manager

First off, if you're on Mac then Homebrew is your goto installation delivery system.

```zsh
brew install asdf
```

Otherwise, we'll need to download the precompiled binary. Note that this is only one way of installing **asdf** [there are several others](https://asdf-vm.com/guide/getting-started.html#_1-install-asdf) but I've found this to be the most reliable. So find the binary based on your current OS from the GitHub releases page: https://github.com/asdf-vm/asdf/releases.

Next, create a new directory called **.asdf** and place the downloaded binary within the directory.

Finally, add the shims to the PATH environment variable

```zsh
export PATH="${ASDF_DATA_DIR:-$HOME/.asdf}/shims:$PATH"
```

Reload your shell with the `reload` alias and type `asdf list`. This should be empty since we haven't added any plugins yet.

If `asdf list` doesn't work or returns a command not found, you might need to add the **asdf** binary to your path.

```zsh
export PATH="${ASDF_DATA_DIR:-$HOME/.asdf}:$PATH"
```

Now obviously this next part depends on the programming languages that you use but for me I primarily use Ruby and Node.

```zsh
asdf plugin add ruby
asdf plugin add nodejs
```

Eventually, the `asdf list` command might look something like below. This showcases the currently active version with an asterisk, along with all locally installed versions.

![Asdf list example](/img/2025/perfect-terminal/asdf-list.png)

Now within your projects you can specify a `.tool-versions` file that contains a plugin name followed by its version (e.g. `ruby 3.4.2`). This tells **asdf** which version to focus the directory on. We can run `asdf install` to install all specified versions from the identified **.tool-versions** file, assuming the asdf plugin is also installed.

You can also directly install versions by typing something like: `asdf install ruby 3.4.2`. Don't know which version to install? Asdf can list all available versions by running: `asdf list all ruby`. When installing / updating dependencies, it can be helpful to ensure the shimmed commands stay up-to-date. A shimmed command, is literally just a wrapper for an executable. Periodically these will need to be reshimmed when you update them. This can be done with the `asdf reshim <name> <version>` command.

Lastly, if you need to set a global fallback version for a programming language you can use the 0.16.x updated `asdf set` command. For example, to set the base level Ruby version we can run `asdf set ruby 3.4.2` which will create a tool-versions file within our home directory.

Now we have version management configured as well!

## Terminal Emulator

For me there are several important features that make for a good terminal emulator:

1. Quick access
2. Tiling
3. Tabs
4. Theming

|        | Fedora (Wayland)                     | Mac OSX                                              | Notes                                                                                                                                                    |     |
| ------ | ------------------------------------ | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| Kitty  | ⛔️ No quick access support in gnome | ✅                                                   | + Cross-platform<br> + Highly customizable<br> + Performant<br> + Easy to backup config<br> + Themes<br> - Quick access drop-down no support for Wayland |     |
| Ddterm | ✅                                   | ⛔️ Gnome only                                       | + Built for gnome<br> + Customizable<br> + Performant<br> - Not as easy to backup config                                                                 |     |
| Tabby  | ✅                                   | ⚠️ Some performance issues on latest version, Slower | + Excellent theming<br> + Customizable<br> - Slower startup<br> - Less performant (especially OSX)                                                       |     |
| iTerm2 | ⛔️ OSX only                         | ✅                                                   | + Built for OSX<br> + Customizable<br> - Many options you likely won't use<br> - Clunky interface<br> feels dated                                        |     |

From the above, there are two winners with similiar characteristics. Those are **Kitty** for Mac OSX and **Ddterm** for Fedora Workstation. Ideally I would love to use Kitty on both archictectures but the inability to use the quick access dropdown in Wayland makes it a no-go for my Linx machine.

### ddterm - Another drop down terminal extension for GNOME Shell

![Ddterm](/img/2025/perfect-terminal/ddterm.png "Image source https://github.com/ddterm/gnome-shell-extension-ddterm")

For Gnome, there is the lovely Gnome Shell Extensions ecosystem along with the [ddterm extension](https://github.com/ddterm/gnome-shell-extension-ddterm). The Shell Extensions page has hundreds of modifications you can
install and further tweak. **ddterm** was built with Gnome and Wayland in mind, and as such works well. These can be directly installed from the Gnome Shell Extensions search page.

In order to configure the quick access hotkey, all that needs to be done is to set the **Toggle Terminal Window** within the **Keyboard Shortcuts** setting pane.

![Ddterm hotkey](/img/2025/perfect-terminal/ddterm-hotkey.png)

### Kitty - The fast, feature-rich, GPU based terminal emulator

![Kitty](/img/2025/perfect-terminal/kitty.svg "Image source https://sw.kovidgoyal.net/kitty/")

[Kitty](https://sw.kovidgoyal.net/kitty/) is my preferred terminal emulator. It has so many options which are controllable with configuration files making it easy to backup. It also looks great from a UI perspective.

![Kitty Example](/img/2025/perfect-terminal/kitty-example.png)

It has tabs, panes, extendable plugins called kittens, and is performant. To install Kitty you need
only run the following command:

```zsh
curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin
```

For Quick Access, there is a kitten called `quick-access-terminal` which is [documented here](https://sw.kovidgoyal.net/kitty/kittens/quick-access-terminal/). Linux and Mac OSX both
require you to specify a window manager hotkey for Kitty.

> **Linux**: "Simply bind the above command to some key press in your window manager"<br> **Mac OSX**: "...go to System Preferences->Keyboard->Keyboard Shortcuts->Services->General and set a shortcut for the Quick access to kitty entry."

## Conclusion

We now have a system configured with all the following tools:

- Shell - ZSH
- Plugin Manager - Antidote
- Version Manager - Mise
- Terminal Emulator - Kitty

What could be better than a perfect terminal? An automated one! I'm in the process of writing a
follow-up post that disects two dotfile managers called RCM and Doot. They both help streamline nearly
all of the above process.

More on those soon! Thanks for reading.
