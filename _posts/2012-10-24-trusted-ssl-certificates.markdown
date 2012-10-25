---
layout: post
title: Enforcing Trusted SSL Certificates on iOS and Mac
category: ios
---

One of the nice things about iOS and Mac networking is that it generally just
works.  In particular, connecting to an SSL-protected host is a transparent
operation for high-level code based on `NSURLConnection`.  As long as the
authenticity of the remote host's public certificate can be verified, the
connection is considered trustworthy.

This broad definition of trust allows connections to arbitrary hosts without
requiring prior knowledge of those hosts' certficates.  This convenience is
generally desirable, but there are times when we want to restrict our trust to
a specific set of known certificates, not just any certificate signed by a
well-known authority or registered with the [Keychain][].

For example, many iOS applications interact with a backend server component.
These connections can be "sniffed" by introducing [proxy software][charles]
between the iOS device and the rest of the network.  While this can be a
wonderful development aid, it highlights the often misunderstood insecurity of
these otherwise "private" communication channels.  This scenario is known as a
[man-in-the-middle attack][mitm].  (It is also how the Siri protocol was
originally cracked.)

Fortunately, Apple includes all of the tools needed for an application to
evaluate the trusthworthiness of individual network connection attempts:

 - [NSURLProtectionSpace Class Reference](https://developer.apple.com/library/ios/#documentation/Cocoa/Reference/Foundation/Classes/NSURLProtectionSpace_Class/Reference/Reference.html)
 - [NSURLAuthenticationChallenge Class Reference](https://developer.apple.com/library/ios/#documentation/Cocoa/Reference/Foundation/Classes/NSURLAuthenticationChallenge_Class/Reference/Reference.html)
 - [Certificate, Key, and Trust Services Programming Guide](https://developer.apple.com/library/ios/#documentation/security/conceptual/CertKeyTrustProgGuide/01introduction/introduction.html)

## Trusted Certificates

To begin, the client application needs its own copy of the trusted server's
public certificate.  This should be in [DER format][DER] (which is just a
binary version of the more familiar [PEM format][PEM]).  To convert a
certificate from PEM to DER format:

    $ openssl x509 -inform PEM -outform DER -in cert.pem -out cert.der

Then add the certificate to the application's resource bundle (or otherwise
encode it in the application binary).  This is safe because it's only the
public portion of the certificate; the private key remains private.

Note that bundling the server's public certificate with the application does
introduce an ongoing maintenance concern.  It will need to be updated whenever
the server's certificate changes (such as after a renewal).  But this is the
simplest approach for the illustrative purposes of this article.

## Evaluating Server Trust

The application now needs a way to determine whether or not the remote host's
certificate matches the bundled certificate.  This is done by establishing a
chain of trust "anchored" on the local certificate.  The server's credentials
are accessed via the `NSURLProtectionSpace` interface.

{% highlight objectivec %}

- (BOOL)shouldTrustProtectionSpace:(NSURLProtectionSpace *)protectionSpace {
    // Load up the bundled certificate.
    NSString *certPath = [[NSBundle mainBundle] pathForResource:@"cert" ofType:@"der"];
    NSData *certData = [[NSData alloc] initWithContentsOfFile:certPath];
    CFDataRef certDataRef = (__bridge_retained CFDataRef)certData;
    SecCertificateRef cert = SecCertificateCreateWithData(NULL, certDataRef);

    // Establish a chain of trust anchored on our bundled certificate.
    CFArrayRef certArrayRef = CFArrayCreate(NULL, (void *)&cert, 1, NULL);
    SecTrustRef serverTrust = protectionSpace.serverTrust;
    SecTrustSetAnchorCertificates(serverTrust, certArrayRef);

    // Verify that trust.
    SecTrustResultType trustResult;
    SecTrustEvaluate(serverTrust, &trustResult);

    // Clean up.
    CFRelease(certArrayRef);
    CFRelease(cert);
    CFRelease(certDataRef);

	// Did our custom trust chain evaluate successfully?
    return trustResult == kSecTrustResultUnspecified;
}

{% endhighlight %}

## NSURLConnection

Integrating the above code with an `NSURLConnection`-based system is easy.
First, the connection delegate needs to advertise its ability to authenticate
"server trust" protection spaces.

{% highlight objectivec %}

- (BOOL)connection:(NSURLConnection *)connection canAuthenticateAgainstProtectionSpace:(NSURLProtectionSpace *)protectionSpace {
    return [protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust];
}

{% endhighlight %}

Then, the connection delegate needs to handle the authentication challenge:

{% highlight objectivec %}

- (void)connection:(NSURLConnection *)connection didReceiveAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge {
    if ([self shouldTrustProtectionSpace:challenge.protectionSpace]) {
        [challenge.sender useCredential:[NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust] forAuthenticationChallenge:challenge];
    } else {
        [challenge.sender performDefaultHandlingForAuthenticationChallenge:challenge];
    }
}

{% endhighlight %}

Trust failures will result in a connection failure:

    The certificate for this server is invalid. You might be connecting
    to a server that is pretending to be "api.example.com" which could
    put your confidential information at risk.

## AFNetworking

[AFNetworking][] implements the `NSURLConnectionDelegate` methods internally.
Fortunately, it also exposes a block-based interface for supplying custom
implementations of the necessary callbacks.  These are applied directly to
`AFHTTPRequestOperation` objects, so a convenient place to set them is in a
custom `HTTPRequestOperation:success:failure:` implementation.

{% highlight objectivec %}

- (AFHTTPRequestOperation *)HTTPRequestOperationWithRequest:(NSURLRequest *)urlRequest
                                                    success:(void (^)(AFHTTPRequestOperation *, id))success
                                                    failure:(void (^)(AFHTTPRequestOperation *, NSError *))failure {
    AFHTTPRequestOperation *operation = [super HTTPRequestOperationWithRequest:urlRequest success:success failure:failure];

    // Indicate that we want to validate "server trust" protection spaces.
    [operation setAuthenticationAgainstProtectionSpaceBlock:^BOOL(NSURLConnection *connection, NSURLProtectionSpace *protectionSpace) {
        return [protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust];
    }];

    // Handle the authentication challenge.
    [operation setAuthenticationChallengeBlock:^(NSURLConnection *connection, NSURLAuthenticationChallenge *challenge) {
        if ([self shouldTrustProtectionSpace:challenge.protectionSpace]) {
            [challenge.sender useCredential:[NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust]
                 forAuthenticationChallenge:challenge];
        } else {
            [challenge.sender performDefaultHandlingForAuthenticationChallenge:challenge];
        }
    }];

    return operation;
}

{% endhighlight %}

[Keychain]: http://en.wikipedia.org/wiki/Keychain_(Mac_OS)
[charles]: http://www.charlesproxy.com/documentation/proxying/ssl-proxying/
[mitm]: http://en.wikipedia.org/wiki/Man-in-the-middle_attack
[DER]: http://en.wikipedia.org/wiki/Distinguished_Encoding_Rules#DER_encoding
[PEM]: http://en.wikipedia.org/wiki/Privacy_Enhanced_Mail
[AFNetworking]: http://afnetworking.com/
