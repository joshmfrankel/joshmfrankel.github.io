---
layout: post
title: Enable ESLint for SublimeLinter while using asdf for version management
category:
- Fixes
---

SublimeLinter is an excellent tool for linting new code quickly and efficiently. Version managers like rvm, nvm, and [asdf](https://github.com/asdf-vm/asdf) are also great tools for smoothly switching between projects with different version requirements. Getting both SublimeLinter and version manager to play nice can sometimes be challenging. I'm going to quickly talk about the simple steps I took to getting ESLint and [asdf](https://github.com/asdf-vm/asdf) to work with SublimeLinter below.
<!--excerpt-->

I'm currently using [asdf](https://github.com/asdf-vm/asdf) and am digging how simple and modular it is. [asdf](https://github.com/asdf-vm/asdf) allows for multiple languages to be managed at once, so instead of having nvm, rvm, etc you just end up configuring a single script. Whatever your version manager is the steps outlined below should work for you.

## Install SublimeLinter-eslint

The first step is to install Sublimelinter-eslint through the Sublime package installation menu. Once done you'll either need eslint installed as a local package or globally. Read the [SublimeLinter-eslint installation](https://github.com/SublimeLinter/SublimeLinter-eslint#installation) section for more details.

## Fix node PATH resolution
With the new SublimeLinter 4, ensuring the node is recognized seems to be currently producing issues. The error I was encountering looks like: <code>SublimeLinter: ERROR: /usr/bin/env: ‘node’: No such file or directory</code>

I first checked <code>which node</code> in console and it returns <code>/home/josh/.asdf/shims/node</code>. This seems accurate since [asdf](https://github.com/asdf-vm/asdf) adds shims to the language binary files. So seems like my PATH is accurate in console but SublimeLinter doesn't recognize it. 

The [current workaround I found online](https://github.com/SublimeLinter/SublimeLinter-eslint/issues/205#issuecomment-370229955) was to explicitly add the <code>.asdf/shims</code> path to the eslint configuration within your <code>SublimeLinter.sublime-settings - User</code> file.

{% highlight json %}
// SublimeLinter Settings - User
{
  "linters": {
    "eslint": {
      "disable": false,
      "args": [],
      "excludes": [],
      "env": {
        "PATH": "~/.asdf/shims:$PATH" // This is the important line
      }
    }
  }
}
{% endhighlight %}

At this point you should start seeing proper linting of JavaScript files which looks something like this:

![SublimeLinter-eslint example](/img/2018/eslint-sublime.png)
