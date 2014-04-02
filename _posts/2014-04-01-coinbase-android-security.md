---
layout: post
title: Coinbase Android Security Vulnerabilities
---

I recently contacted [Coinbase](https://www.coinbase.com) about some security vulnerabilities in their [Bitcoin Wallet](https://play.google.com/store/apps/details?id=com.coinbase.android) and [Coinbase Merchant](https://play.google.com/store/apps/details?id=com.coinbase.android.merchant) apps via their [white hat program](https://coinbase.com/whitehat). Sadly, they disagreed with the security issues I brought to their attention. Fortunately, these issues are very easy to resolve and I have strongly urged them to do so. I am disclosing them here to alert the public of these security risks and so their users will can take necessary action protect their money.

## SSL Certificate Verfication
Coinbase [wisely recommends](https://coinbase.com/docs/api/authentication#security) that all clients of their API should validate the SSL certificate presented to prevent MITM attacks. However, they fail to do this in their own Android applications.

This opens up the possibility for someone to use SSL Pinning (installing an additional certificate authority on the Android device) to enable a MITM attack. This can be used to view their *client_id* and *client_secret*. It is very easy to install an additional certificate authority on a device. This is trivial to perform using a tool like [Charles Proxy](http://www.charlesproxy.com/).

This also leaves the open the possibility of a spoofed SSL certificate being presented by the MITM. A spoofed certificate could be a SSL Certificate with a valid signing chain whose root siging certificate authority is already installed on the device. For example, a certificate from Verisign instead of from DigiCert (Coinbase's actual certificate authority). This type of SSL attack first published by [Moxie Marlinspike](http://www.thoughtcrime.org/papers/null-prefix-attacks.pdf). In this case, no additional certificate authority would need to be installed on the victim's device.

## API Key Security
Coinbase has failed to adequately protect their application's API *client_id* and *client_secret*. They are published in the [source code on GitHub](https://github.com/coinbase/coinbase-android) and visible during the authentication process if a [man in the middle attack](http://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM) is established, which I've outlined above.

Here is the POST request to https://coinbase.com/oauth/token during the auth process. The *client_id* and *client_secret are clearly visible*.

```
POST /oauth/token HTTP/1.1
Content-Length: 302
Content-Type: application/x-www-form-urlencoded
Host: coinbase.com:443
Connection: Keep-Alive
User-Agent: Apache-HttpClient/UNAVAILABLE (java 1.4)

client_id=34183b03a3XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXf5&client_secret=2c481f46fXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX22d&grant_type=authorization_code&redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob&code=764f56XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX968600

```

... and here is the response. Once the attacker has beaten the SSL connection, they can view the *access_token*.

```
HTTP/1.1 200 OK
Server: cloudflare-nginx
Date: Wed, 02 Apr 2014 04:02:42 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Set-Cookie: [Redacted]
Cache-Control: no-store
Etag: "eda4841ff7d50a67a6b1facca8018527"
Pragma: no-cache
Status: 200 OK
Strict-Transport-Security: max-age=31536000
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Rack-Cache: invalidate, pass
X-Request-Id: 63349570-36d9-413a-b9cb-31912f85ca04
X-Runtime: 0.029784
X-Ua-Compatible: IE=Edge,chrome=1
CF-RAY: 114a21e0dc560502-SEA

{"access_token":"d1aXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX947","token_type":"bearer","expires_in":7200,"refresh_token":"3ae1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX37caedc","scope":"all"}

```

## Putting It All Together
Beause an attacker has a valid *client_id*, valid *client_secret*, and the ability to defeat the application's SSL connection, requests can be viewed, repeated, and modified by an attacker. The attacker can also use these vulnerabilities make API requests at a later time using an *access_token* stolen during a authentication response.

### Modifying
With a *client_secret* exposed, requests can be resigned with a valid signature ([documentation here](https://coinbase.com/docs/api/authentication#hmac)).

### Repeating
At the time of writing, the Coinbase Android applications **do not** make use of a *nonce* or *request signatures*, so it is not even necessary for SSL to be compromised in order to repeat requests. Even if Coinbase had made use of them, generating a new *nonce* and resigning the request is trivial.

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
* Validate SSL connections in your applcation as outlined [here](http://developer.android.com/training/articles/security-ssl.html) and as suggested in your [documentation](https://coinbase.com/docs/api/authentication#security).
* Make use of your new API which supports nonce and request signing.

## Timeline of Disclosure
* 2014-03-11 - Vulnerabilities first disclosed to coinbase via whitehat@coinbase.com
* 2014-03-14 - Follow up email sent
* 2014-03-14 - Julian Langschaedel of Coinbase responds and dismisses the SSL issues. Does not address others.
* 2014-03-14 - Two more follow up emails are sent to whitehat@coinbase.com and Julian in regards to issues not responded to.
* 2014-04-01 - The final draft of this post is sent to whitehat@coinbase.com to give them another attempt to address this internally before it is disclosed to the public.