---
layout: post_page
title: Signing Git Commits and Tags Using Your GPG Key
comments: true
description: How to configure Git signing using GPG on OSX
---

Signing Git commits using GPG isn't a requirement in most projects. It is however a nice to have feature, especially in cases where it is important for you to have your identity as the commiter verified. This prevents, for instance, an impersonator from going undetected when they submit a commit as you. It's in Ona's best interest to verify the identity of contributors linked to us in the numerous data collection projects we contribute code to, mainly to prevent this [totally plausible horror story](https://mikegerwitz.com/papers/git-horror-story) presented by Mike Gerwitz from happening.

Here are instructions on how to set up GPG signing on OSX. Those of the 'Linux master race' should be able to easily follow.

Make sure you have gpg2 installed by running:

    which gpg2

If the path to the gpg2 binary isn't displayed you can install it using Homebrew:

    brew install gpg2

### 1. Create a GPG Keypair

This step should only be done by those who don't already have a GPG keypair.

If you want to do it the right way, sub-keys and all, I recommend you use [this guide](https://alexcabal.com/creating-the-perfect-gpg-keypair/). However, if you just want a goddamn keypair, use [GitHub's guide](https://help.github.com/articles/generating-a-new-gpg-key/).

### 2. Add GPG Support to Git

Now you'll need to configure Git to use your GPG private key for signing. First get your key ID by running:

    gpg2 --list-secret-keys|grep sec

You should see two output lines; the first showing the path to the file holding the key, and the second the key details (including the key ID--which is what you want). The format for the second line should be something like:

    sec    [Key Length]/[Key ID] [Date Created and Expiry Date]

Now add your key ID to Git and tell it to use the gpg2 binary for signing:

    git config --global user.signingkey [your key ID]
    git config --global gpg.program gpg2

Test if Git can use the private key by amending your last commit in a project:

    git commit -s -S --amend

The *-s* switch appends your name and email address to the bottom of the commit message (called signing-off a commit) while *-S* signs the commit using your GPG private key.

You can make Git sign all commits and tags by default by running:

    git config --global commit.gpgsign true
    git config --global tag.gpgsign true

### 3. Cache GPG Passphrase Using GPG Agent

If you don't want to type your private key's passphrase every time you sign a commit, you can use [gpg-agent](https://wiki.archlinux.org/index.php/GnuPG#gpg-agent) to temporarily cache the passphrase.

Install gpg-agent and [pinentry](https://www.gnupg.org/related_software/pinentry/index.en.html) (used for entering the passphrase) by running:

    brew install gpg-agent pinentry-mac

Make sure the following lines exists in *~/.gnupg/gpg.conf*:

    default-key [your key ID]
    use-agent

Create a new configuration file *~/.gnupg/gpg-agent.conf* and add the following lines:

    use-standard-socket
    default-cache-ttl 60
    write-env-file /Users/[your user]/.gnupg/gpg-agent-info
    pinentry-program /usr/local/bin/pinentry-mac

Confirm that the path to pinentry-mac is the one specified above (modify if need be) by running:

    which pinentry-mac

You should also change the value of *default-cache-ttl* to the number of seconds you want the passphrase to be kept valid.

Add the following lines to your user's shell profile file e.g *~/.bash_profile*:

    export GPG_TTY=$(tty)
    [ -f ~/.gnupg/gpg-agent-info ] && source ~/.gnupg/gpg-agent-info
    if [ -S "${GPG_AGENT_INFO%%:*}" ]; then
        export GPG_AGENT_INFO
    else
        eval $( gpg-agent --daemon --options ~/.gnupg/gpg-agent.conf --write-env-file ~/.gnupg/gpg-agent-info )
    fi

Then reload the profile file:

    killall gpg-agent
    . ~/.bash_profile

Test to see if gpg-agent can cache your private key's passphrase by amending your last commit:

    git commit -s -S --amend

Pinentry should pop up as soon as you save the commit message. All subsequent commits shouldn't ask for the passphrase until it expires on gpg-agent.

### 4. Add Your GPG Public Key to GitHub

Copy your GPG public key into your clipboard:

    gpg2 --armor --export [your key ID] | pbcopy

Go to your GitHub [key settings](https://github.com/settings/keys), click on 'New GPG Key', and paste whatever is in your clipboard into the text entry field. All commits and tags you've signed should now have a 'VERIFIED' sticker on GitHub 😀.