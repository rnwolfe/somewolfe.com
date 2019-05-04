---
author: rn.wolfe
date: 2017-09-05 01:42:41+00:00
draft: false
title: Cisco ISE and Client Certificate Chain with Any Purpose EKU
type: post
url: /2017/09/05/cisco-ise-and-client-certificate-chain-with-any-purpose-eku/
featuredImage: /images/ise-eku-test-sub.png
categories:
  - Cisco ISE
  - Troubleshooting
tags:
  - authentication
  - certificates
  - Cisco ISE
  - EAP
  - TLS
  - troubleshooting
  - x509v3
---

I recently came across quite an interesting issue during a Cisco ISE implementation using EAP-TLS. This was using EAP-FAST to perform EAP Chaining using the Cisco AnyConnect NAM module; however, the inner method was EAP-TLS and that’s where the problem resided. My authentication was failing due to “unsupported certificate in client certificate chain.”

Ultimately, the problem was that the certificate the client was authenticating with was sending its certificate chain along with its authentication request. In the chain, one of the Certificate Authority certificates contained an Extended Key Usage (EKU) attribute of “Any Purpose”. Note that Enhanced and Extended Key Usage are used interchangeably by most references. The EKU field is sometimes also known as the Enhanced Key Usage or Application Policies field – mostly in Microsoft lingo. Whenever this attribute was present in the client’s certificate chain, the authentication would fail due to “unsupported certificate in client certificate chain.”

The only resolution that I could find is to use a different CA without the “Any Purpose” EKU value. If you have a multiple-tier PKI, you can simply issue another Subordinate without the value specifically for client authentication certificates (assuming the Root is not the one with the problematic EKU). I would expect that this is an uncommon occurrence given I’ve only run into it once over the years. In fact, the default Subordinate CA template on a Microsoft PKI implementation does not contain EKUs, so you would have to go out of your way to include that for a specific purpose.

Use of EKUs in a CA certificate doesn’t even really seem to be valid practice. RFC 5280, which covers x509v3 Digital Certificates, specifies that EKU attributes should only be used on “end entity” certificates.

<blockquote>2.1.12. Extended Key Usage

This extension indicates one or more purposes for which the certified public key may be used, in addition to or in place of the basic purposes indicated in the key usage extension. In general, this extension will appear only in end entity certificates.</blockquote>

End entities would be the systems that certificates are issued to for usage in certificate-enabled functionalities such as SSL, TLS, authentication, digital signature, etc. and are not used for issuing certificates. Certificate Authorities issue end entity certificates, and are therefore not end entities themselves. The EKU field specifies for what purposes those end entity certificates should be used. For example, “Client Authentication” and “Server Authentication” are EKU values which are commonly used for EAP-TLS. All a CA really needs are the standard Key Usage values of Certificate Sign, Digital Signature, Key Encipherment, etc.

The only purpose I can see where “Any Purpose” would make itself into a CA certificate is if whoever stood it up intended to use the issuing certificate as an end entity certificate, as well. For instance, if I am hosting a web site on my CA server to provide my CRL and CA chain via HTTPS, I could add the “Server Authentication” purpose to my issuing certificate; however, that shouldn’t really be the way it’s done. To best protect the integrity of your issuing certificate, it should be only used for that purpose. If you need an SSL certificate for that site, it would be best to issue yourself a client certificate from the issuing CA certificate specifically for that purpose rather than share it for multiple purposes.

To validate this, I created a new three-tier PKI. The Offline Root CA was used to issue an Issuing CA and neither of those certificates had any EKU and used the standard templates in a Microsoft PKI. I then issued a Test Subordinate CA from the Issuing CA that contained the “Any Purpose” EKU. This chain can be seen below:

{{< smallimg src="/images/ise-eku-cert-chain.png" alt="Test PKI Path" >}}

You can see the EKU of the Test Subordinate CA contains the "Any Purpose" EKU:

{{< smallimg src="/images/ise-eku-test-sub.png" alt="Test Sub EKU"  width="70%">}}

I attempted an EAP-TLS authentication using the Test Subordinate CA and received a failure due to the unsupported certificate. I then re-issued a certificate from the standard Issuing CA and the authentication was the same. The only difference between the two scenarios was the use of the “Any Purpose” EKU.

There is an ongoing TAC case exploring whether this should be a bug or is expected behavior. It is currently pending the Cisco ISE Business Unit’s input. I will update this post when more information is provided.

**Update on 10/11/2017**: A bug has been opened for this "documentation" bug - [CSCvg10726](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCvg10726). The developers provided input to the TAC case that states that this is expected behavior. This suggests there will be no change to this in the future, so keep an eye out for this EKU in CAs being used in client authentications or when building new CAs for client authentications.
