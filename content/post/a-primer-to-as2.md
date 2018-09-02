+++
title = "A Primer to AS2"
date = 2015-05-15T02:09:00Z
updated = 2015-05-24T12:16:03Z
tags = ["AS2", "Mule"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

*This post was reproduced on [PortX's blog](https://portx.io/what-is-as2/)*.

AS2 has been out for a while but I bet most of you have never heard of the [spec](http://www.ietf.org/rfc/rfc4130.txt). Indeed, 
a couple of months ago I myself was oblivious to it. That is until [ModusBox](http://www.modusbox.com/) asked me to help 
them build a first-class AS2 connector for [Mule](http://www.mulesoft.com/platform/soa/mule-esb-open-source-esb).

AS2 (Applicability Statement 2) is a protocol that operates over HTTP/s and is typically 
used between organisations to exchange business data in any format securely. The exchange of 
business data for the purposes of purchasing goods and services is commonly referred to as B2B. 
Traditionally, B2B was done over [VANs](https://en.wikipedia.org/wiki/Value-added_network) (i.e., Value-added Networks) that were secure and reliable 
however also expensive. The Internet provided a cheaper option. Yet HTTP is not sufficiently secure. 
Now you might be saying “You have HTTPs for transporting sensitive data". Well for one thing, 
HTTPs protects your data while it's transiting from point-to-point; not end-to-end. It's possible 
that a document intended for your partner passes through an HTTPs intermediary such as a 
gateway owned by a third-party:

![Gateway](/images/gateway.png)

After the gateway's HTTPs software peels off the request's security layers, the document is 
open for reading and modification with potential harmful consequences. End-to-end security 
requires a higher level protocol and this is where AS2 comes into play [1]. AS2 relies on 
[S/MIME](http://www.ietf.org/rfc/rfc3851.txt) to offer encryption and signing capabilities protecting the document throughout its 
entire journey across the network from unauthorized recipients and tampering. Moreover, 
digital signatures give confidence to the recipient that the sender is really who it claims 
to be. AS2 processes for securing the data and reading it are depicted as follows:

![AS2 send and receive](/images/as2-send-receive.png)

For an AS2 sender to secure the document, it:

1. Signs the document with its own private key
2. Generates a session key and encrypts the document along with the digital signature using the generated session key
3. Encrypts the session key with the receiver's certificate and includes the key in the request

On the other end, the AS2 receiver:

1. Decrypts the included session key with its own private key
2. Uses the session key to decrypt the document
3. Verifies the document's signature using the originator's certificate.

Note the certificates are usually exchanged out-of-band, as part of setting up a bilateral 
trading partner agreement. Consequently, AS2 normally requires some manual setup unlike a 
protocol, say, like TLS/SSL.

Having solid evidence that your partner actually received the document you sent is another 
good reason why one might want to transport documents on top of AS2. Consider the gateway example. 
You send a purchase order for 1000 widgets over HTTPs to the manufacturer. An encoding bug 
in the gateway software causes it to add an additional zero to the order amount:

![Gateway error](/images/gateway-error.png)

You get a 200 OK HTTP status code confirming the order was processed so you assume the 
manufacturer received the order as it was sent. The next month, to your surprise, you 
find 10,000 widgets delivered to your doorstep. Quite reasonably, you refuse to settle 
the bill complaining the number of widgets is substantially more than the amount you ordered. 
The manufacturer, also quite reasonably, demands payment for the widgets it shipped to you. 
Given that no party has irrefutable evidence contradicting the other, there is no easy way 
in general to find out who should bear the responsibility for this disagreement.

Disagreements like the one just mentioned are extremely improbable, if not impossible, to 
happen when businesses are communicating over a fully secure AS2 channel due to the use of 
MICs and receipts as well as digital signatures. Below is high-level view of how AS2 proceeds 
to achieve this state of affairs without encryption:

![AS2 process](/images/as2-process.png)

Before sending an HTTP POST containing the document, the AS2 sender:

1. Calculates a digest, the MIC (Message Integrity Check), over the document and records it for later use
2. Signs the document with its own private key.

Once the AS2 receiver has the document, it:

3. Verifies it with the sender's public key
4. Creates a receipt, called an MDN (Message Disposition Notification) in AS2 lingo, confirming the document was received and whether it was successfully verified.
5. Re-calculates the MIC over the received document and adds it to the MDN
6. Signs the MDN with its own private key and sends it to the sender

The sender verifies the MDN with the partner's public key and compares its recorded MIC 
against the receipt's MIC.

Let's revisit the previous example. Given the above steps, should the gateway modify the 
purchase order, then the manufacturer would immediately know that the order was altered 
during transit because a signature mismatch would occur. In such a scenario, the manufacturer 
would return to the sender an MDN saying the document failed verification. Even if the manufacturer 
accidentally or maliciously changed the ordered amount after delivering a successful MDN, 
the manufacturer would be forced to claim responsibility. The reason is simple: the manufacturer 
sent an MDN to you acknowledging receipt of the purchase order in addition to its content. 
This property where the manufacturer cannot deny receiving the order as sent by you is known 
as non-repudiation (or NRR in short). AS2, in its full glory, can help provide NRR of origin and receipt.

Till now I've discussed the security aspects of AS2. What about efficiency? AS2 has an option 
such that you can have the receiver deliver an MDN asynchronously as opposed to delivering 
it in the same HTTP session as the sender's original message. This option is specified as 
an AS2-specific HTTP header and it provides two major benefits:

1. Reduces the risk of an HTTP response timeout. This is especially of value when decrypting and verifying large documents.

2. The receiver processes the document at its own pace. The implication is fewer resources can be allocated to the receiver for decrypting and verifying the data since the HTTP response time is no longer a crucial factor.

Below we have an illustration of how async MDNs work given the document is encrypted and signed:

![AS2 process for signing and encrypting](/images/as2-process-signed-encrypted.svg)

As you can see above, the recipient replies with a 200 OK as soon as it receives the request. 
The HTTP response only acknowledges that a request was received; not that its content was 
decrypted or verified. Asynchronously, the recipient goes on to decrypt and verify the document. 
Once it's done, it:

1. Creates an MDN
2. Reads the URL from the AS2-specific HTTP header in the request specifying the destination for the async MDN
3. Posts an HTTP request carrying the MDN to the URL.

In addition to async MDNs, compression is another efficiency feature in AS2's arsenal. When 
signing is applied, compression can happen before or after the data is signed. Benefits exist 
to both ways of doing things but I'll leave that discussion for another blog post.

This concludes a 10,000 foot-view of AS2. Considering its security, non-repudiation, and 
efficiency features, you can see why AS2 is the way to go for doing business over the Internet. 

<div style="text-align: justify; line-height: 1.3;">
  <span style="font-family: Times, Times New Roman, serif; font-size: small;">
    <span class="num">1: WS-Security is another way where end-to-end security can be provided. But while AS2 is format agnostic, WS-Security is SOAP specific.</span>
  </span>
</div>