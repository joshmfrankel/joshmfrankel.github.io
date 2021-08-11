---
layout: post
title: Signing commits for Github and Sublime Merge
categories:
- tutorials
tags:
- Git
- Github
- Sublime Merge
- GPG
- Linux
---

Signing commits is a great way to add additional level of confidence to your code.
This is especially important if you are an open source contributor. By
signing your commit you're saying that it originated from a verified author.
This is accomplished by using GPG which is a free encryption and signing tool.

Github has taken this one step farther and now shows signed commit authors with
a verified badge. Not only is this a great way to see at a glance if code comes
from a trusted source, but the verified badge looks slick.
<!--excerpt-->

For the remainder of this tutorial, I'm assuming you don't have a generated GPG key. If
you do (or aren't sure) you can run the command:
`gpg --list-secret-keys --keyid-format LONG`. This will list out all available
GPG keys with public and private keys. If you'd like to use an existing
private/public key pair, go ahead and skip down to the step, <a href="#finding-your-generated-gpg-key">Finding your GPG key"</a>.

<del>I'm also writing this tutorial from the perspective of a Linux distribution but
it should work relatively the same for MacOS (not sure about Windows).</del> I've now
updated this tutorial for both Linux Debian and MacOSX.

## Making sure GPG is installed

{% include blockquote.html quote="GnuPG allows you to encrypt and sign your data and communications; it features a versatile key management system, along with access modules for all kinds of public key directories." title="GPG (GNU Privacy Guard)" source_link="https://gnupg.org/" %}

Most variants of Debian should have <acronym title="GNU Privacy Guard">GPG</acronym>
installed by default. You can check this by running `which gpg` which should return
the installation location.

If it doesn't then you need to install it with `sudo apt-get install gpg` for linux or `brew install gnupg` for homebrew on MacOS.

## Generating a new GPG key

Github has some great [documentation on generating gpg keys](https://help.github.com/en/articles/generating-a-new-gpg-key).
<del>Unfortunately, the command it recommends for gpg as well as gpg2 doesn't actually work. What I found works
is the following</del>:

``` terminal
gpg --gen-key
```

**UPDATE 2020**: Upon running through this process on MacOS, I've found that `gpg --full-generate-key` works appropriately. For Linux
based distributions you may still need to use the old --gen-key method above.

We're going to run through what the process will look like step-by-step.

For the first prompt, **just hit enter**. This will select the default option. The default of RSA and RSA is what we want.

``` shell
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection?
```

Github recommends the maximum of 4096 so let's use that. **Type out 4096**.

``` shell
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
```

Next it will ask you if you want the key to expire. We don't, so **use the
default by pressing enter**.

``` shell
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
```

Time for a confirmation. Is the above information correct? **Type "y" to continue**.

``` shell
Is this correct? (y/N) y
```

Now we need to identify ourselves. It wouldn't make sense for a signed commit
to be without identity verification would it?

First step is to **enter your full name** as you'd like it to appear in the signature.
I've listed my name out but please use your own (we don't need any more Josh Frankel's
running amok).

**NOTE**: This name should match the one stored in your `~/.gitconfig` user name key.

``` shell
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: Josh Frankel
```

{% include blockquote.html quote="When asked to enter your email address, ensure that you enter the verified email
address for your GitHub account. To keep your email address private, use your
GitHub-provided no-reply email address." source_link="https://help.github.com/en/articles/generating-a-new-gpg-key" source_text="Github" %}

Next you'll need to **use the same email address that you use for Github**. This is
really important to ensure that your GPG signature matches the username. I believe
it is also how Github determines whether or not to place the verified badge next
to your commits.

``` shell
Email address: josh@tutorial.com
```

You can leave Comment blank by **pressing enter**.

``` shell
Comment:
```

Next you'll get an identity confirmation. If all the below is correct then
**type O for Okay**.

``` shell
You selected this USER-ID:
    "Josh Frankel <josh@tutorial.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit?
```

The next two sections deal with passphrase prompting at the command line for Debian
and MacOSX respectively.

### Debian GPG passphrase

Now it will prompt you to **enter a passphrase** which for me looked like this (it may
be a command line prompt for other distributions and operating systems):

![Access prompt](/img/2019/access-prompt.png "Access Prompt")

You'll be prompted to re-enter this to confirm. Don't forget this password as it
is necessary for signing commits. Think of it as logging into the GPG identity.
I saved mine in a password manager.

### MacOSX GPG Passphrase
Ideally, you should receive a passphrase prompt at this point. If not then you most likely don't have one configured. While
not really clear, you can add the **pinentry-mac** program.

```
brew install pinentry-mac
```

Next you'll need to inform gpg-agent of pinentry-mac with the following append
command:

```
echo "pinentry-program /usr/local/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
```

After which if you're still not able to sign commits you may need to restart
your gpg-agent.

```
pkill gpg-agent
gpg-agent --daemon
```

I believe that this is due to gpg-agent running as a daemon which means that it can't
prompt you to enter a passphrase. Instead it has to rely on pinentry-mac to send
the passphrase prompt.

### Finish key generation

After the identity confirmation, the last step is to just do random work on your computer until the GPG key is
marked as trusted

``` shell
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

gpg: key 3AA5C34371567BD2 marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   2  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 2u
pub   4096R/3AA5C34371567BD2 2019-06-01
      Key fingerprint = X72D 8C40 70ZB 73A2 EFAE  981C ED3A 021E 7160 C89A
uid                  Josh Frankel <josh@tutorial.com>
sub   4096R/42B317FD4BA89E7A 2019-06-01
```

That's it we've created our first GPG key. Nice job!

## Finding your generated GPG Key

Now we need to grab our generated GPG key and add it to our Github account. We
can do this with a couple commands.

`gpg --list-secret-keys --keyid-format LONG` will list out all your existing keys that have a public and private
pairing.

``` shell
gpg --list-secret-keys --keyid-format LONG

/home/josh/.gnupg/secring.gpg
-----------------------------
sec   4096R/3AA5C34371567BD2 2019-06-01
uid                          Josh Frankel <josh@tutorial.com>
ssb   4096R/42B317FD4BA89E7A 2019-06-01
```
Now, we need to export our GPG key to be used on Github via the command: `gpg --armor --export <key-id>`

{% include blockquote.html quote="Since PGP can operate both in ASCII mode and “raw” mode, it’s important to understand when to use which one. When sending something that will be viewed as text (i.e. an email), you should use ASCII mode. On the other hand, when sending a file, you can (and should, it will make the encrypted file smaller) use the default non-ASCII mode. To operate in ASCII mode, use the --armor (or -a) switch." title="ASCII vs Raw mode" source_link="https://theprivacyguide.org/tutorials/gpg.html" source_text="The Privacy Guide" %}


First, grab the key id which can be found on the `sec` line after the keysize specification
of `4096R`. In our case that means using **3AA5C34371567BD2** from above.

Using the key id, we'll gather the GPG key signature by using the `--armor` flag which is
useful for seeing keys in ASCII format. Without this we won't
be able to copy our key and paste it into Github. Additionally, the `--export`
flag simply exports the key that we specify.

These must be used in tandem in order to gather proper output.

``` shell
gpg --armor --export 3AA5C34371567BD2
```

This will output your GPG key to your console. Make sure to copy the entire thing
starting with the `BEGIN` comment and ending with the `END` comment.

``` shell
-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: GnuPG v1

Lots of characters and lines of text
Lots of characters and lines of text
Lots of characters and lines of text
-----END PGP PUBLIC KEY BLOCK-----
```

Let's add our key to github!

## Adding your GPG key to Github

At this point we can follow [Github's guide directly which can be found here](https://help.github.com/en/articles/adding-a-new-gpg-key-to-your-github-account). It has some great screenshots and
step-by-step instructions. I won't repeat them here. Once you're done move onto the next
section below for learning how to configure git for automatic signing.

## Automatically signing commits in Git

We've added our GPG key to Github and the key matches our Github email address,
we can configure git to start using it.

This is done by setting our `user.signingkey` to match our GPG key id.

``` shell
git config --global user.signingkey 3AA5C34371567BD2
```

Running this command, will add a new entry to **~/.gitconfig** which looks like:

``` shell
[user]
  email = ...
  name = ...
  signingkey = 3AA5C34371567BD2
```

Now by default Git will use the specified signingkey for signing commits.

Commits are signed with the format of `git commit -S` [[More on signing commits here]](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work#_signing_commits). Now, if you're like me remembering to type out
the `-S` flag every time is more mental gymnastics that you're willing to do. Let's make it automatic so that
we don't even need to think about it. We can do this by specifying a new configuration
option of **true** for `commit.gpgsign`.

 ``` shell
git config --global commit.gpgsign true
```

If we take a look at our **~/.gitconfig** configuration we'll see another new
entry:

``` shell
[commit]
  gpgsign = true
```

Now the first time you go to sign a commit it will prompt you with entering a
passphrase for the GPG key. We did this above after the identity verification step
of generating a GPG key. If you have an existing key you'll need to enter your
created passphrase here as well.

Can't remember it? You can always create a new
one to use by following the [Generating a new GPG key](#generating-a-new-gpg-key) section of
this tutorial.

My prompt allowed me to remember my passphrase which means I no longer need to enter it.
I really like how simple this makes the signing process. Let me know if this isn't the
case for operating system or if you have a different solution.

On the flip-side, there's something to be said about always entering the passphrase for every commit.
This is a bit more secure if you consider the scenario of someone having access to your physical computer
then a saved passphrase means it isn't really you.
That being said I'm not too worried about something like this as I'm using several other security
features on my laptop.

Enough exposition!

Once you start signing commits you can see the signatures appear next to your
commit headers in the log using `--show-signature`. Neat!

``` shell
git log --show-signature

# Log output
commit xxxxxxxxx
gpg: Signature made Wed 29 May 2019 09:30:37 AM EDT using RSA key ID 3AA5C34371567BD2
gpg: Good signature from "Josh Frankel <josh@tutorial.com>"
Author: Josh Frankel <josh@tutorial.com>
Date:   Wed May 29 09:30:37 2019 -0400

    Did some stuff and signed a commit. Yay!
```

And if you push a newly signed commit to Github you can see your verified identity
badge in all its glory.

![Verified Github commit](/img/2019/verified-commit-github.png "Verified Github commit")

The badge is what it's all about.

## Enabling Signing of commits in Sublime Merge or other Git clients

Signing of commits works on the command line beautifully now. We can alleviate
any concerns of identity as now all of our commits are signed and verified. But
what about external Git tools?

I prefer to use Git with a dedicated client. Having a
Git client makes things like adding hunks or fixing conflicts a breeze. There
are a ton of different ones out there but I've found that **Sublime Merge** is fast,
simple, and integrates well into my existing Sublime Text workflow. (I'm a Sublime fan
if you couldn't tell.) [It's also free to try out just like Sublime](https://www.sublimemerge.com/)

The problem with external tools is that they don't use the command line and therefore
work differently than running `git commit -S` directly. In other words, when I first tried to commit via Sublime Merge
I got the following cryptic error:

![Sublime Merge signing error](/img/2019/sublime-merge-error.png "Sublime Merge signing error")

The tip off point for me was the first line of the error message that mentioned
**/dev/tty**. This is just a reference to your current terminal.

``` shell
gpg: cannot open tty `/dev/tty': No such device or address
```

So what can we do here?

Well we can specify that we want to run GPG in **no-tty** mode.
This means that the terminal won't be used for output or prompting. This is perfect
for external applications.

To accomplish this we'll need to add **no-tty** to our **gpg.conf** configuration.
By doing so we can keep our process automated and ensure freedom of committing in our
favorite external tools.

Open **~/.gnupg/gpg.conf** and add `no-tty` to the end of the file. I added a comment
to explain why the option was in there so that future me remembers.

``` shell
# You may also list arbitrary keyservers here by URL.
#
# Try CERT, then PKA, then LDAP, then hkp://subkeys.net:
#auto-key-locate cert pka ldap hkp://subkeys.pgp.net

# No tty disables terminal prompts allowing Sublime Merge to work
no-tty
```

Now go back to Sublime Merge and try committing again. Viola, it works!

![GPG Hooray Gif](/img/2019/gpg-hooray.gif "GPG Hooray Gif")

One downside of adding no-tty to our **gpg.conf** is that trying to run gpg on the
command line stops working because we just disabled tty (duh). If for some reason
you need to use gpg you can always go back in and comment out the `no-tty` line
on your **gpg.conf** file. I
haven't been able to find another way sign commits in a Git tool
while not disabling gpg for the command line.

![Sublime Merge signing success](/img/2019/sublime-merge-success.png "Sublime Merge signing success")

## Conclusion

So to quickly wrap up here's what we accomplished:

* We generated a new GPG Key
* We exported the generated key
* We pasted the key into Github to match our account to GPG identity
* We configured signing of commits to be automatic
* We ensured that Git clients worked with signing commits

With that git committing and as always I'd love to hear your questions and feedback
in the comments. Thanks for reading.

