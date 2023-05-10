---
title:  "Multi Factor Authentication For The Win (or is it?)"
date:   2022-11-22 12:34:48 +0300
categories: security
author: "Can"
excerpt: "Learn about the basics of MFA"
header:
    overlay_image: /assets/images/mfa.jpeg
    overlay_filter: linear-gradient(rgba(2, 0, 36, 0.5), rgba(0, 146, 202, 0.5))
slug: mfa-for-the-win
sidebar:
    nav: docs
layout: custom
---

In today's digital age, ensuring the security of our online accounts and sensitive information is more crucial than ever. 
One of the most effective ways to achieve this is through Multi Factor Authentication (MFA), a security measure that requires users to provide two or more independent credentials to verify their identity.
MFA has become increasingly popular as it significantly reduces the risk of unauthorized access, even if one of the authentication factors is compromised, though there are ways to 
bypass the effects of MFA.

## Why MFA?

The most common way to authenticate a user is by using a username and password. A password with some degree of randomness and length could potentially save you from calculated brute force attacks, however solely
relying on a password is not enough to protect your account.

First of all, passwords can be guessed and I'm sure there are still people out there who use easy-to-guess passwords such as `123456` or `password`. Humans are not `random` by nature
and we all tend to attribute something we can remember to our passwords, which makes them vulnerable to brute force attacks.

Secondly, even if the password length is relatively large, it can still be cracked by using a dictionary attack.
In fact, according to [Specops Software Weak Password Report](https://specopssoft.com/wp-content/uploads/2022/02/Specops-Software-Weak-Password-Report-2022-2.pdf) published in 2022, 93% of real password attacks use passwords with 8 or more characters.

These dictionary attacks can be prevented to a certain degree via implementing account lockout and security monitoring measures, to track subsequent loging attempts to a certain account and lock it if the number of attempts exceeds a certain threshold.
However, this approach is not foolproof either, as `Password Spray Attacks` can try many password combinations one at a time, slowly and methodically, to avoid detection. In fact, [Akamai
reported that](https://www.darkreading.com/edge/credential-stuffing-attacks-behind-30-billion-login-attempts-in-2018) they encountered 115 million password spraying attempts on average.

That is why, as an additional layer of security to protect your account, you should consider using MFA, however it is not a silver bullet either.

## Authentication Factors

When it comes to MFA, there are three main authentication factors:

- **What You Know**: Something you know, such as a password, PIN, or security question.
- **What You Have**: Something you have, such as a mobile phone, smart card, or security token.
- **What You Are**: Something you are, such as a fingerprint, voice, or facial recognition.

For example, by possessing a debit card (what you have) and typing a PIN (what you know), you’ve utilised two factor classes for one authentication process.
Mixing factor classes leads to better security because it is unlikely that a criminal could compromise both authentication mechanisms.

So, MFA is simply a combination of two or more of these factors. For example, when you log in to your bank account, you may be asked to enter your password (what you know) and then enter a code sent to your mobile phone (what you have).

## Types of Multi Factor Authentication

**In-Band MFA:** As part of the authentication process, you would also have to transmit the generated code after a successful username - password combination is typed.

So, if an attacker manages to steal your credentials, they can also steal the OTP, since it is transmitted over the same channel.

**Out-of-band auth:** Separates the channels of authentication such that even if your primary credentials are stolen, an attacker can’t easily capture the second factor.

If your bank, calls your phone after you try to log into your online account, this is an example of an out-of-band second factor auth. 

A criminal may be able to control your computer, but unless they’ve also stolen your mobile phone or hijacked your call forwarding, they are not able to influence the authentication process in this scenario.

## OTPs

OTPs (One-Time Passwords) are a randomly generated string of characters that are only valid for a single login session or transaction and are only predictable between the subject and the authentication system. 
Once it is used, the OTP will never be generated or used again, hence it is called `one-time`. Thus, even if an attacker manages to capture a particular OTP, it will never work again on any other session.

However, there are certain limitations to OTPs, the most important being the size of the code. Depending on the size of the code, the number of possible combinations will vary. For example,
a 6 digit code would have a probability set of 10^6, which is 1 million codes and if we are using a time based OTP method like `TOTP`, we would typically have a validity window such as `30 seconds`, although you can set this time window to any value you'd like.
If that is the case, all million codes could be shown in `347 days`, assuming that the codes are never repeated. So, if an expired code is captured by an attacker, it could potentially be used in future authentication requests, given the possibility of collision.

Also, OTP's are calculated based on a shared `secret` or `seed` value. 
This secret is a random string of characters that is known by both the user and the service provider. 
The secret is used to generate a unique OTP for each login attempt. The service provider can then verify the OTP against the secret to authenticate the user. If this secret is not random enough,
it could potentially be guessed by an attacker, which would render the OTP useless.

Let's take a look at the different types of OTPs:

**1 - Seed Value Based OTP`s**

To establish a certain degree of randomness in the OTP, a seed value is used. This seed value is a random but unchanging number which is shared among the authentication parties.
This seed is leveraged to generate a seemingly random and frequently changing OTP that can be computed by either party independent of each other.
Each party does their own calculations and compares the results to determine if the OTP is valid. If the OTP's match, the authentication is successful.

**2 - HMAC Based OTP’s**

This OTP method uses an incremented value or a counter as the seed value and is based on the HMAC algorithm.
An HMAC algorithm uses a strong hash and a trusted calculation algorithm along with a pre-shared key (seed value) in a way to generate unique outputs for unique supplied content.
Both parties should ensure that the counter value should change and not repeat itself. If the counter value is repeated, the OTP will be the same and the authentication will fail.
Also, the counter value should be in sync between the parties, otherwise the OTP will not match. An example of this type of OTP is `HOTP` (RFC 4226).

**3 - Time Based OTP’s**

Time-based one-time password (TOTP) authentication solutions involve a current date and/or time value in the counter calculation. As the time changes, so
too does the counter value, usually at fixed predetermined intervals, like every 30 seconds to 10 minutes.
The clock or the time source in both parties should be trusted and in sync. If the time is not in sync, the OTP will not match and the authentication will fail.
To stay away from the hussle of converting each time to the time zone of the authenticating party, the time is usually converted to `UTC` before the calculation. The time is further
converted into a timestamp value such as `Unix Time` or `POSIX Time` before the calculation. An example of this type of OTP is `TOTP` (RFC 6238).

A TOTP is always updating based on time all the time, and older TOTP values automatically invalidate more frequently based on time. Therefore, HOTP OTP values are potentially
valid in much longer periods of time and therefore considered riskier than TOTP values.

## MFA Mediums

The OTP can be delivered to the user in a variety of ways. The most common methods are SMS, email, and authenticator apps, although there are other methods such as phone calls and hardware tokens.

**1 - SMS**

- The most common form of MFA.
- The user enters their username and password, and then receives a text message with a code that they must enter to complete the login process.
- SMS is not the most secure form of MFA, as it is vulnerable to SIM swapping attacks.

**2 - Email**

- Similar to SMS, but instead of receiving a text message, the user receives an email with a code that they must enter to complete the login process.
- Email is also not the most secure form of MFA, as it is vulnerable to phishing attacks.

**3 - Authenticator Apps**

- Authenticator apps are a more secure alternative to SMS and email.
- Instead of receiving a code via SMS or email, the user receives a code via an authenticator app such as Google Authenticator or Authy.
- Authenticator apps are more secure than SMS and email, however they are vulnerable to malware attacks. For example, a criminal could install malware on your phone that intercepts the OTPs generated by the authenticator app. This is why it is important to keep your phone secure and only install apps from trusted sources.

**4 - Biometrics**

- Biometrics are a more secure alternative to SMS, email, authenticator apps, and hardware tokens.
- Instead of receiving a code, the user must provide a biometric such as a fingerprint or facial scan to complete the login process.
- Biometrics are more secure than SMS, email, authenticator apps, and hardware tokens however, they are vulnerable to spoofing attacks. For example, a criminal could use a high resolution photo of your face to bypass facial recognition.

## Common MFA Attacks

**1 - Phishing**

Phishing is a common attack that involves a lookalike man-in-the-middle proxy website that acts as the real site the victim is intending to visit. In this attack, every request by the victim is
proxied to the attacker's server and everything from the real server is proxied to the user.

Here is an [attack](https://www.youtube.com/watch?v=xaOX8DS-Cto&t=307s) where the attacker sends a mock invitation pretending to be sent from `LinkedIn` to spoof the session cookie of the victim.

**2 - Poor OTP Creation**

You should never rely on custom or not so well known OTP generation methods. Instead, you should use a well-known and trusted library to generate your OTPs. For example, you could use the `pyotp` library to generate your OTPs.
Typically, the OTP generation algorithm should involve:

- A secure hash algorithm
- A random & secret seed value that forever remains a secret between the involved parties

**3 - Poor OTP Storage**

You should never store your OTPs in plain text. Instead, you should store them in a secure database or a secure file.
Please read about the [RSA Breach](https://www.wired.com/story/the-full-story-of-the-stunning-rsa-hack-can-finally-be-told/) for an example of what can happen if your codes are compromised.

**4 - OTP Replay**

All OTP codes surpassed by newer OTP codes should no longer be valid! Some OTP types such as `TOTP` includes timestamps in the calculation to ensure that the OTP is only valid for a certain period of time.
If the OTP codes do not expire, an attacker can record the OTP codes and use them at a later time to bypass the authentication system.

**5 - OTP Brute Force**

When OTP codes are typed in, the authentication system should have an “account lockout” feature that prevents attackers from guessing over and over again unsuccessfully.
If the attacker is able to guess the OTP code without any limitations, they have a higher chance bypassing the authentication system.

## Conclusion

MFA is a powerful security measure that can significantly reduce the risk of unauthorized access. However, it is important to understand that MFA is not a silver bullet. It is still possible for an attacker to bypass MFA, for example by using social engineering to trick the user into providing their credentials and OTP. Therefore, it is important to use MFA in conjunction with other security measures such as strong passwords, password managers, and security awareness training.

## References

- Inspired by this book: [Hacking Multifactor Authentication by Roger A. Grimes](https://www.google.com.tr/books/edition/Hacking_Multifactor_Authentication/Zpv9DwAAQBAJ?hl=en&gbpv=1&printsec=frontcover)
- RFC 4226 - HOTP (HMAC-Based One-Time Password Algorithm) [https://tools.ietf.org/html/rfc4226](https://tools.ietf.org/html/rfc4226)
- RFC 6238 - TOTP (Time-Based One-Time Password Algorithm) [https://tools.ietf.org/html/rfc6238](https://tools.ietf.org/html/rfc6238)
- RFC 6287 - OCRA (OATH Challenge-Response Algorithm) [https://tools.ietf.org/html/rfc6287](https://tools.ietf.org/html/rfc6287)

So this was all for today! Enjoy the rest of your day!