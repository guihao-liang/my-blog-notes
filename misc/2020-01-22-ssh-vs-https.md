---
title: Push commits without typing password
subtitle: git url using ssh vs https
---


```bash
git remote add
```

In my mac, I don't have to type password to push changes to remote. That's because that OSX key-chains caches my username and password for github.

On my mac, I use to have ssh-agent running in order to do the authentication for me.

```bash
eval ssh-agent -s
ssh-add ~/.ssh/id_rsa
```

I thought `ssh-agent` will delegate me to encrypt and decrypt authentication data for me. But recently, my colleagues told me they never used such thing before. SSH will use the private key automatically for them.

But why I need it? I set password for my private key whereas my colleagues they use empty password phrase for their private keys.

> As a security measure, most people sensibly protect their private keys with a passphrase, so any authentication attempt would require you to enter this passphrase. This can be undesirable, so the ssh-agent caches the key for you and you only need to enter the password once, when the agent wants to decrypt it (and often not even that, as the ssh-agent can be integrated with pam, which many distros do).<br>
> The SSH agent never hands these keys to client programs, but merely presents a socket over which clients can send it data and over which it responds with signed data. A side benefit of this is that you can use your private key even with programs you don't fully trust.

cited from [what's the purpose of ssh-agent?](https://unix.stackexchange.com/questions/72552/whats-the-purpose-of-ssh-agent)