---
title: SSH and Password Managers
date: 2020-01-24
tags: [Pass, SSH]
---

The ssh program on Linux allows you to specify a program or script from where it should get a passphrase when it needs one. This allows you to, for instance, hookup it up to your password manager to feed it the passphrase. You can do this by specifying the path to the program to execute in the `SSH_ASKPASS` environment variable.

I use the [pass](https://www.passwordstore.org/) password manager (which installs the `pass` command). Check it out, it's pretty awesome. Here's how to hook up `pass` to ssh:

First, create a simple bash script to feed `pass` the path to the password in my password store that I want it to display (let's say I save it in `~/ssh-pass-passphrase.sh` and forward the input to it):

```sh
#!/bin/bash

read pathToPassphraseWithinPass
# Calling pass with the path to the secret will output the secret's value 
/usr/bin/pass $pathToPassphraseWithinPass
```

Then, make sure your user is the only one allowed to edit the script:

```sh
chmod 0700 ~/ssh-pass-passphrase.sh
```

Now set the `SSH_ASKPASS` environment variable:

```sh
export SSH_ASKPASS=~/ssh-pass-passphrase.sh
```

You should now be able to use it with any ssh command that needs a passphrase (e.g `ssh-add`) like this:

```sh
echo path/to/passphrase/within/pass | ssh-add ~/.ssh/key-with-passphrase
```

Please note that this will not work if the ssh command is called directly on the terminal (is attached to a terminal). A workaround is to call it after a pipe. In the command above, I've used the pipe to pass the path to the passphrase in `pass` to `ssh-add`. `ssh-add` will eventually call `~/ssh-pass-passphrase.sh` and provide the path when `read` is fired in the script.

Here's text from the ssh manual detailing what `SSH_ASKPASS` does:

> If ssh needs a passphrase, it will read the passphrase from the current terminal if it was run from a terminal.  If ssh does not have a terminal associated with it but DISPLAY and SSH_ASKPASS are set, it will execute the program specified by SSH_ASKPASS and open an X11 window to read the passphrase.  This is particularly useful when calling ssh from a .xsession or related script.  (Note that on some machines it may be necessary to redirect the input from /dev/null to make this work.)
