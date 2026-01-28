---
title: "gpg commit signing"
date: 2026-01-27T20:00:00-08:00
draft: false
summary: "learning how commit signatures work and getting verified badges on github."
---

i set up gpg commit signing to learn how cryptographic signatures work in practice. also, verified commits show your github profile picture in the commit history, which looks pretty cool.

## what it does

when you sign a commit with gpg, you're using public key cryptography to prove the commit came from whoever holds your private key.

you generate a key pair. the private key stays on your machine and signs commits. the public key goes to github. anyone can verify your signature using that public key.

when you commit, git hashes the commit contents and encrypts that hash with your private key. that's the signature. github decrypts it with your public key and compares it to the commit hash. if they match, the signature is valid.

## the trust model

asymmetric cryptography means you have two related keys. anything encrypted with the private key can be decrypted with the public key.

when you sign a commit, you encrypt a hash with your private key. anyone with your public key can decrypt it and verify the hash matches. if it does, they know the commit was signed by whoever holds that private key.

the security relies on keeping your private key secret. the public key can be shared freely.

## what it protects against

if someone gets your github token, they can push commits. without signing, those commits look like they came from you. with signing enabled, unsigned commits get marked as unverified.

if someone compromises github, they could modify commit history. signed commits would fail verification if tampered with.

if someone gets your laptop, they can set git config to your name. those commits appear as yours locally. but they can't sign them without your private key.

the protection isn't perfect. if someone has your laptop and your gpg key isn't protected by a passphrase, they can use it. even with a passphrase, a keylogger could capture it.

signing doesn't prove the code is good. it only proves who authored it.

## setup

generate a key with `gpg --full-generate-key`. export the public key and upload it to github. configure git to sign automatically.

```bash
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true
```

after that, every commit gets signed. no extra steps.

one gotcha is the email address. the email in your gpg key must match a verified email on your github account. otherwise github won't show the verified badge.

## why i wanted to learn this

i work in security, so understanding how cryptographic identity works matters. reading about asymmetric crypto is one thing. actually setting it up and seeing how github validates signatures is another.

details like how they cache public keys, how they handle multiple emails on one key, what causes verification to fail. you only learn these by doing it.

plus, verified commits show your github profile picture next to them in the commit history. small detail, but it makes the history look more polished.

## key management

the private key is critical. if someone gets it, they can sign commits as you. if you lose it, you need to generate a new one.

you can add a passphrase for extra security, but you'll need to enter it on every commit. for a personal machine with disk encryption, no passphrase is fine.

backing up the key matters. export with `gpg --export-secret-keys` and store it somewhere safe. that way you can restore it on another machine.

## end result

once set up, signing happens automatically. every commit gets a verified badge on github with your profile picture.

the real value is understanding how the signatures work. how they're created, what gets validated, how the trust chain functions. useful knowledge for anyone working with authentication systems.
