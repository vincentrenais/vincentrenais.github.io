---
layout: post
title: "Public key pinning as a great alternative to certificate pinning"
description: "Let's dig into public key pinning and see why it's probably a better solution than certificate pinning in most cases."
og_image: 
tags: [ios, security]
---

Following up on my previous [post](http://www.vincentrenais.com/posts/adding-certificate-pinning-to-your-ios-app) about certificate pinning, let's look at public key pinning as an alternative.

I mentioned in the conclusion of my last post that one of the caveat of certificate pinning was dealing with the certificate replacement. Public keys don't expire and therefore can be re-used to issue new certificates. By pinning the key we remove the need for a certificate replacement strategy.

Extracting the key is a little more work but it is really worth it and is probably the best solution for most apps.
 
#### Extracting the key

We'll start from where we left off in the previous post, just download the starter project on [Github](https://github.com/vincentrenais/CertificatePinningExample/tree/master).

First we need a method to extract the key.

{% highlight swift %}
private func extractPublicKeyFromCertificates(_ certs: [SecCertificate]) -> [SecKey] {
        var trust: SecTrust?
        let policy = SecPolicyCreateBasicX509()
        var publicKeys: [SecKey] = []
        for cert in certs {
            let status = SecTrustCreateWithCertificates(cert, policy, &trust)
            var key: SecKey?
            if status == errSecSuccess {
                guard let finalTrust = trust else { return [] }
                key = SecTrustCopyPublicKey(finalTrust)
            }
            guard let publicKey = key else { return [] }
            publicKeys.append(publicKey)
        }

        return publicKeys
    } 
{% endhighlight %}

We pass the array of certificates as a parameter and we are returning an array of `SekKey` objects.

The security framework is written in C, we don't get nice instance methods, instead we'll have to work with functions. It's going to hurt a little but we'll get there.

First we create an empty array of `SecKey` objects.

Then we create a `SecTrust` variable called `trust`, we will pass this object's pointer to one of the functions later on.

We also create a `SecPolicy` object using the `SecPolicyCreateBasicX509()` function. This will return an object that conforms to the X509 public key infrastructure standard.

Then, for each certificate in the array we are calling  `SecTrustCreateWithCertificates()`. We pass the certificate, the policy and the pointer to the `SecTrust` object previously created. The function will return a `OSStatus` with failure or success as well as the `SecTrust` object that will be stored in the previously provided pointer.

Finally we can check the `OSStatus`, and if `errSecSuccess` is returned we can then create a `SecKey` with the `SecTrustCopyPublicKey()` function to which we pass the returned  `SecTrust`.

#### Using the key

The changes required in the `enableCertificatePinning()` are minimal.

{% highlight swift %}
private func enableCertificatePinning() {
    let certificates = getCertificates()
    let publicKeys = extractPublicKeysFromCertificates(certificates)
    let trustPolicy = ServerTrustPolicy.pinPublicKeys(
        publicKeys: publicKeys,
        validateCertificateChain: true,
        validateHost: true)
...
{% endhighlight %}

First we load the certificates, then we extract the public keys. The `ServerTrustPolicy` needs to be modified as well, we are now using `pinPublicKeys()` instead of `pinCertificates()`.

That's it, we are now pinning the public key from the embedded Apple certificate instead of the certificate itself.

## Conclusion

In most cases public key pinning is probably the best solution, you just have to make sure the same public key is used when issuing new certificates on the server your app is talking to. One of the caveat would be that the key is static which may violate key rotation policies. No solution is ever perfect...

_The final Xcode project is available on [Github](https://github.com/vincentrenais/CertificatePinningExample/tree/publicKeyPinning)_