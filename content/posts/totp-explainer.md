---
title: "TOTP Made Simple: How App Codes Work"
date: 2025-02-14T10:00:00+08:00
draft: false
author: "Sven"
summary: "A plain-English guide to time-based one-time passwords and why they keep logins safer."
showtoc: true
tags: ["security", "totp", "2fa"]
---

# TOTP Made Simple: How App Codes Work

If you have ever logged in and been asked for a 6-digit code from an app like Google Authenticator, Microsoft Authenticator, or Authy, you have used **TOTP**. TOTP stands for **Time-based One-Time Password**. It is a simple idea that adds a strong extra lock to your account.

This post explains TOTP in plain English: what it is, how it works, and why it is safer than relying on passwords alone.

## What is TOTP?

TOTP is a short code that changes every few seconds (usually every 30 seconds). You type that code after your password. Because the code keeps changing, a stolen code quickly becomes useless.

Think of it like a hotel key card that automatically expires in half a minute. Even if someone copies it, it only works for a brief moment.

## Why not just use a password?

Passwords can be guessed, leaked, or reused. TOTP adds a second proof that you are really you. Even if someone knows your password, they still need the rotating code from your device.

This is why TOTP is often called **two-factor authentication (2FA)**:
- **Something you know:** your password.
- **Something you have:** your phone app that generates the code.

## How does TOTP actually work?

At setup time, the website and your authenticator app share a **secret key**. It is usually shown as a QR code. Your app scans it once and stores it.

After that, both sides can generate the same code at the same time:

1. They look at the current time.
2. They combine the time with the secret key.
3. They run a math formula to generate a 6-digit number.

Because the website and your app use the same secret and the same clock, they produce the same code for a short time window. The server checks whether the code you type is the expected one for that time window.

> In short: **shared secret + current time = temporary code**.

## Why does the code expire so quickly?

The quick expiration is the safety feature. If a code is intercepted, it only works for a very short time. Most systems allow a tiny “grace window” (for example, 30 seconds before or after) in case your phone’s clock is slightly off.

## What happens if I lose my phone?

Because the code lives on your phone, losing it can lock you out. That is why most services provide **backup codes**. Store those in a safe place (like a password manager). You can also register multiple devices if the service allows it.

## Is TOTP the same as SMS codes?

Not exactly. SMS codes are also one-time passwords, but they are delivered over the phone network, which can be intercepted or redirected. TOTP codes are generated locally on your device, which makes them **less vulnerable to SIM swap attacks**.

## Common questions

### Does TOTP require internet?
No. The app uses the current time and the secret key, so it works offline.

### Why are TOTP codes usually 6 digits?
Six digits strike a balance between usability and security. There are 1,000,000 possible codes, which is plenty for a short time window.

### Can I use the same TOTP app for many sites?
Yes. The app can store multiple secret keys, one per website or service.

## Quick takeaway

TOTP is a simple but powerful security layer. It works because **your phone and the website share a secret and use time to generate a short-lived code**. The result is a login step that is easy for you but hard for attackers to bypass.

If you can enable TOTP on your important accounts, it is one of the best security upgrades you can make with just a few minutes of setup.
