---
layout: post
title: Coinbase Android Security Vulnerabilities
---

I recently contacted [Coinbase](https://www.coinbase.com) about some security vulnerabilities in their Coinbase Bitcoin Wallet](https://play.google.com/store/apps/details?id=com.coinbase.android) and [Coinbase Merchant](https://play.google.com/store/apps/details?id=com.coinbase.android.merchant) apps via their [white hat program](https://coinbase.com/whitehat). Sadly, they disagreed with the security issues I brought to their attention. Fortunately, these issues are very easy to resolve and I have strongly urged them to do so. I am disclosing them here to alert the public of these security risks and so their users will can take necessary action protect their money.

## API Key Security
Coinbase has failed to adequately protect their application's API *client_id* and *client_secret*. They are published in their application's [source code on GitHub](https://github.com/coinbase/coinbase-android) and visible during the authentication process if a [man in the middle attack](http://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM) is established which I will outline below.

## SSL Certificate Verfication
Coinbase [wisely recommends](https://coinbase.com/docs/api/authentication#security) that all clients of their API should validate the SSL certificate presented to prevent MITM attacks. However, they fail to do this in their own Android applications.

This opens up the possibility for someone to use SSL Pinning (installing an additional certificate authority on the Android device) to enable a MITM attack. This can be used to view their *client_id* and *client_secret*. It is very easy to install an additional certificate authority on a device. This is trivial to perform using a tool like [Charles Proxy](http://www.charlesproxy.com/).

I have yet to confirm this, but theoretically it also leaves the open the possibility of a spoofed SSL certificate being presented by the MITM. This type of SSL attack first published by [Moxie Marlinspike](http://www.thoughtcrime.org/papers/null-prefix-attacks.pdf). In this case, no additional certificate authority would need to be installed on the victim's device.

## Putting It All Together
Beause an attacker has a valid *client_id* , valid *client_secret*, and the ability to defeat the application's SSL connection, requests can be viewed, repeated, and modified by an attacker. The attacker can also use these vulnerabilities make API requests at a later time using an *access_token* stolen during a authentication response.

### Modifying
With a *client_secret* exposed, requests can be resigned with a valid signature ([documentation here](https://coinbase.com/docs/api/authentication#hmac)).

### Repeating
At the time of writing, the Coinbase Android applications **do not** make use of a *nonce* or *request signatures*, so it is not even necessary for SSL to be compromised first in order to repeat requests. Even if Coinbase had made use of them, generating a new *nonce* and resigning the request is trivial.

### Potential Harm
* Without the SSL connection being compromised, simply repeating requests could allow an attacker to repeat payments to an address, requests to sell bitcoin, or requests to buy bitcoin.
* With a compromised SSL connection, an attacker could gain full control of a user's account.

## Recommendations to Coinbase Users
* Discontinue using the [Coinbase Bitcoin Wallet](https://play.google.com/store/apps/details?id=com.coinbase.android) and [Coinbase Merchant](https://play.google.com/store/apps/details?id=com.coinbase.android.merchant) apps until Coinbase uses a new *client_id* and *client_secret* and begins to verify SSL certificates in its applications.
* Revoke access to the Coinbase apps granted [here](https://coinbase.com/account/applications).
* Check your account for any suspicious transactions or settings.

## Recommendations to Coinbase
* Issue new a new *client_id* and *client_secret* for your applications and keep these confidential.
* Use code obfuscation (e.g. ProGuard) to hide important keys if their APK is decompiled.
* Validate SSL connections in your applcation as outlined [here](http://developer.android.com/training/articles/security-ssl.html).