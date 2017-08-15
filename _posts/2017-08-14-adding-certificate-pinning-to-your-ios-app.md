---
layout: post
title: "Adding certificate pinning to your iOS app with Alamofire"
description: "Make your app more secure and avoid man-in-the-middle attacks with certificate pinning."
og_image: 
tags: [ios, security]
---

I received an email the other morning from [';--have i been pwned?](https://haveibeenpwned.com) that read like this:

> You're one of 85,176,234 people pwned in the Dailymotion data breach

So I went ahead and deleted my account. Not sure why I had an account to begin with. One thing that struck me was the fact that I was not surprised at all. Just another data breach. These things became the norm, your data will get stolen at some point.

So what can we do to avoid this as much as possible in our beloved iOS apps? Certificate pinning is one answer.

I'm no security expert but one thing I know is that certificate pinning is like putting a lock on your door, it is not foolproof but you would have to be insane not to have one.

#### Why?

It's simple, we rely more and more on outside services such as your favorite Starbucks wifi network or the subway fancy new free wifi. These are great, but since the security of HTTPS connections relies on trust (trusting the chain of certificates all the way back to the root certificate). A proxy and a rogue root certificate and anybody can intercept the communications between your app and the server. Well, not "anybody" but you get my point.

#### How?

Certificate pinning removes the need for trust. By bundling the server certificate with your app and check, for every request, that the certificate in your app matches the one on the server, you limit greatly the potential for attacks.

#### Does it hurt?

A little bit, there's some C functions involved as most of the security framework is written in C. But it should be fine.

#### No networking library
 
If you are not using Alamofire, I'd like to point out to [Florian Kugler](https://twitter.com/floriankugler) and [Rob Napier](https://twitter.com/cocoaphony) great [video](https://talk.objc.io/episodes/S01E57-certificate-pinning) on [Swift Talk](https://talk.objc.io). They go through the certificate pinning implementation with `NSURLSession`. I highly recommend getting a Swift Talk subscription by the way. Great stuff!

## Alamofire

You can download the starter project on [GitHub](https://github.com/vincentrenais/CertificatePinningExample/tree/starterProject). The [Alamofire](https://github.com/Alamofire/Alamofire) pod is already included so you should be able to run the app out of the box.

We are starting with a very simple app that loads a webpage in a WebView. Below is the code for `ViewController.swift`.

{% highlight swift %}

import UIKit
import Alamofire

class ViewController: UIViewController {

	@IBOutlet weak var webView: UIWebView!

	override func viewDidLoad() {
	    super.viewDidLoad()

	    let url = URL(string: "https://www.apple.com")!
	    Alamofire.request(url).responseData { result in
	    if let data = result.data, let requestURL = result.request?.url {
	        self.webView.load(data,
	                          mimeType: result.response?.mimeType ?? "",
	                          textEncodingName: result.response?.textEncodingName ?? "",
	                          baseURL: requestURL)
	        }
	    }
	}
}

{% endhighlight %}

If you run the app at this point you should see Apple's website in a WebView.

Now let's add certificate pinning to this app.

#### Step 1 - Make it fail

The best way to check if it is secure is to make it fail. We'll start there.

The iOS Security framework format for certificates is `SecCertificate`. We are going to implement certificate pinning with an empty array of `SecCertificate` objects and make sure the requests do not go through.

As of now we are using `Alamofire.request` to send a request to the server. This is a nice convenient method that uses a default instance of `SessionManager` configured with the default `URLSessionConfiguration`. To implement certificate pinning we need to create our own `SessionManager`.

{% highlight swift %}

@IBOutlet weak var webView: UIWebView!

private var sessionManager: SessionManager?

override func viewDidLoad() {

...

{% endhighlight %}

What we need now is a method to enable certificate pinning. Let's write it.

{% highlight swift %}
private func enableCertificatePinning() {
  let certificates: [SecCertificate] = []
  let trustPolicy = ServerTrustPolicy.pinCertificates(
    certificates: certificates,
    validateCertificateChain: true,
    validateHost: true)
  let trustPolicies = [ "www.apple.com": trustPolicy ]
  let policyManager =  ServerTrustPolicyManager(policies: trustPolicies)
  sessionManager = SessionManager(
    configuration: .default,
    serverTrustPolicyManager: policyManager)
}
{% endhighlight %}

* First we are creating an empty array of type `SecCertificate`. Remember we want it to fail.
* The `ServerTrustPolicy` is an object that evaluates the server trust with a given set of criteria to determine whether the server trust is valid and the connection should be made. It is recommended to validate both the chain of certificates and the host so these two parameters should be set to true.
* `trustPolicies` is a dictionary with the aforementioned `ServerTrustPolicy` object as the value and the host as the key, in our case `www.apple.com`. We'll use it to instantiate a `ServerTrustPolicyManager`.
* `policyManager` is an instance of `ServerTrustPolicyManager`. It takes a dictionary of policies as parameters. We need this object to instantiate our `SessionManager`. 
* Finally we instantiate our `SessionManager` with a `.default` configuration and our `policyManager`.

Now let's call this method from `viewDidLoad` and replace `Alamofire.request` with `sessionManager?.request`.

{% highlight swift %}
override func viewDidLoad() {
  super.viewDidLoad()

  let url = URL(string: "https://www.apple.com")!
  enableCertificatePinning()
  sessionManager?.request(url).responseData { result in
{% endhighlight %}

We are all set, Alamofire will check our certificate against the host and the server certificate before letting any request go through. 

Ok, it's time to check, run the app and make sure Apple's website does NOT load.

The output in the debug console should look something like this:

{% highlight shell %}
HTTP load failed (error code: -999 [1:89]) for Task <6DCDA5EE-0F9C-4E30-B553-E5E4614E2502>.<1>
{% endhighlight %}

#### Step 2 - Fix it!

What we need first is the server certificate from `www.apple.com`. We will use the OpenSSL command line to achieve this.

{% highlight shell %}
openssl s_client -connect www.apple.com:443 -servername www.apple.com < /dev/null | openssl x509 -outform DER > appleCert.cer
{% endhighlight %}

* `s_client` - the SSL client from OpenSSL
* `-connect` - sub-command to connect to an SSL server
* `www.apple.com:443` - server address + port
* `-servername` - in case the server is hosted on a shared hosting service
* `< /dev/null` - to avoid a prompt
* `x509` - name of the format of certificates
* `-outform` - to specify the format we want it in
* `DER` - pretty common binary format when dealing with certificates
* `> appleCert.cer` - the end file we want

Now you should have a file named `appleCert.cer`. Let's add it to our Xcode project. Drag and drop the file into Xcode. Make sure you select `copy items if needed` and that you check the target in the `add to targets` section when you get to this screen:

{% include image.html path="documentation/xcode_file_import_confirm_prompt.png" path-detail="documentation/xcode_file_import_confirm_prompt@2x.png" alt="Xcode File Import Confirm Prompt" %}

To make sure the file was added to the main target:
* Right click on the `.cer` file inside the Xcode project navigator and select `Show File Inspector`
* Check the last section at the bottom called `Target Membership` and make sure the main target is checked.

Now that we have the certificate in the project, let's write a function to load it and cast it to the right format. The format we need is `SecCertificate`.

{% highlight swift %}

private func getCertificates() -> [SecCertificate] {
	let url = Bundle.main.url(forResource: "appleCert", withExtension: "cer")!
	let localCertificate = try! Data(contentsOf: url) as CFData
	guard let certificate = SecCertificateCreateWithData(nil, localCertificate)
	   else { return [] }

	return [certificate]
}	    

{% endhighlight %}

We force unwrap using `!` twice in this snippet of code, even though this is something I usually try to avoid, in this case it makes sense as we actually want the app to crash if `url` and/or `localCertificate` are `nil`. Perhaps we did not import the certificate file properly or maybe it wasn't added to the main target. In any case we want to know.

Now let's use our new method inside `enableCertificatePinning()` :

{% highlight swift %}

private func enableCertificatePinning() {
    let certificates = getCertificates()
    let trustPolicy = ServerTrustPolicy.pinCertificates(
...
{% endhighlight %}

Great, now run the app and everything should be back to normal. Congrats, you just implemented certificate pinning.

## Conclusion

#### Caveat
The main caveat is the certificates' expiration date. The server certificate expires so it will be replaced at some point and when that happens, if your app is not ready with the new one, well you have a problem... So you need to adopt some sort of a strategy where you update your server certificate regularly (let's say every year) whatever the expiration date. You include the old certificate as well as the new one, which is why we are using an array of certificates. This way you are giving your users enough time to update before the old certificate expires.

#### Public Key Pinning
In a future post we'll see how to implement public key pinning, which is a way to partly remove the aforementioned caveat.

_The final Xcode project is available on [Github](https://github.com/vincentrenais/CertificatePinningExample/tree/master)_