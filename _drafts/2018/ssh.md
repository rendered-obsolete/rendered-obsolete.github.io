---
layout: post
title: Migrating from MEF1 to MEF2
tags:
- ssh
- git
---

[Bitbucket](https://developer.atlassian.com/blog/2016/04/different-ssh-keys-multiple-bitbucket-accounts/)
[Github general authentication docs](https://help.github.com/categories/authenticating-to-github/)
[Github gist](https://gist.github.com/jexchan/2351996)

On Windows, check `%USERPROFILE%\.ssh\` (e.g. `c:\Users\jake\.ssh`)

```
Host jeikabu-github
    HostName github.com
    User git
    IdentityFile ~/.ssh/jeikabu-github
```

In terminal window run `ssh -T jeikabu-github` to connect to _Host_ specified above, and output should be similar to:
```
Hi jeikabu! You've successfully authenticated, but GitHub does not provide shell access.
```

If there's also `key_load_public: invalid format` there's a [good Stackoverflow answer](https://stackoverflow.com/questions/42863913/key-load-public-invalid-format) you probably used __Save public key__ button in PuTTY Key Generator.  That will create a public key like:
```
---- BEGIN SSH2 PUBLIC KEY ----
Comment: "rsa-key-20180813"
AAAAB3NzaC1yc2EAAAABJQAAAgEA41Rlq76Nc3yqyC6qtEgRpm3ebOpwXBmowawk
1zT1jxIeIle0mvsoF6qXA/P1b8p2ASSOKlbuFibDzZo/NCJFd7BA71pBEZvRHu+B
<SNIP>
Qeb2i6GLCiO/1gYED0He1RB8NLrIfITWWstf8Z21kePfcbFNOPfZB9A3LI7wBGsZ
zKQyCmFkhVHkasgdih/Wod8zVf9ud3DnDIulHnEvrwDa0tYJlp9W/MW17E8ZHo2z
nzGJ3jU=
---- END SSH2 PUBLIC KEY ----
```

Commits were being attributed to the wrong user.  Change user and email address for local repository (instead of using `--global`):
```
git config user.name Jake
git config user.email Jake@jake.com
```
