---
title: 'Multiple SSH Keys for Git and Using 1Password SSH Agent'
date: 2023-08-06T09:00:00-00:00
draft: false
---

You might have the need to use multiple SSH keys for your Git repositories. For example, you might have a personal GitHub account and a work GitHub
account. I got to this exact situation and this is how I solved it.

I presume you have two different SSH keys stored in `~/.ssh`. You can use the combination of `core.sshCommand` setting and `includeIf` conditional
include statement to achieve this.

You can set your personal SSH key as the default one and then override it for your work account. You can do this by adding the following to your
`~/.gitconfig` file:

```
[user]
  name = Jakub Janecek
  email = your-github@email.com
  signingkey = ssh-ed25519 <public-key-hash>
[core]
  sshCommand = ssh -i ~/.ssh/personal_key
[commit]
  gpgsign = true
[gpg]
  format = ssh
[includeIf "gitdir:~/work/"]
  path = ~/work/.gitconfig_work
```

I also recommend enabling signing of your commits. If you also [upload your SSH key to your GitHub account](https://github.com/settings/keys) for
signing it will show your commits as **verified**. You can use the same SSH key for both authentication and signing.

And now let's see how to override the default SSH key for your work projects. You can do this by creating a new file `~/work/.gitconfig_work` with the
following content:

```
[user]
  name = Jakub Janecek
  email = your-email@work.com
  signingkey = ssh-ed25519 <public-work-key-hash>
[core]
  sshCommand = ssh -i ~/.ssh/work_key
[commit]
  gpgsign = true
[gpg]
  format = ssh
```

The only limitation is that all your work projects must be under the `~/work` directory but I presume that's quite reasonable.

## Bonus: Using 1Password SSH Agent

![1Password SSH Key Authorization Prompt](/images/20230806_1password.png)

I am using 1Password as my password manager and I noticed I can start using it to store my SSH keys as well. I can then use the 1Password SSH agent to
load the keys into the SSH agent. The benefit is that the keys don't have to be loaded into the SSH agent all the time (accessible to all processes!)
but only when I need them and when approved by 1Password authentication (e.g., Touch ID on my Mac or using my Apple Watch). I also don't have to worry
about them when reinstalling my computer. Follow 1Password [documentation](https://developer.1password.com/docs/ssh/agent) for more information but
essentially you can do this:

#### `~/.ssh/config`

```
Host *
  IdentityAgent "~/Library/Group Containers/<1password-path>/t/agent.sock"
```

#### `~/.gitconfig`

```
[user]
  name = Jakub Janecek
  email = your-github@email.com
  signingkey = ssh-ed25519 <public-key-hash>

[core]
  sshCommand = ssh -i ~/.ssh/personal_key.pub

[commit]
  gpgsign = true

[gpg]
  format = ssh

[gpg "ssh"]
  program = "/Applications/1Password.app/Contents/MacOS/op-ssh-sign"

[includeIf "gitdir:~/work/"]
  path = ~/work/.gitconfig_work
```

Notice that we are using the public part of your key for the `core.sshCommand` setting and the `gpg.ssh.program` setting points to 1Password SSH
agent. You can get both of these from 1Password application.
