# Troubleshoot LetsEncrypt Connectivity

Many people encounter issues with the LetsEncrypt process due to internet connectivity problems.

The following is an overview of common causes, and ways to troubleshoot.

## Common Causes

### ACME Challenge Authorizations Fail Due to Geo-Blocking

To get a certificate from LetsEncrypt, their ACME Server server must prove that you (the Subscriber) controls a domain.

* For `HTTP-01` and `TLS-ALPN-01` challenges, LetsEncrypt will attempt to connect to your server *from multiple vantage points around the world* to load a unique challenge url.  *All attempts must be successful.*  This means your domain **MUST** be connected to the public internet. Furthermore, your domain **MUST** allow unobstructed traffic to the `/.well-known/acme-challege` directory on port 80 for the `HTTP-01` challenge, or **MUST** allow unobstructed traffic to port 443 (`TLS-ALPN-01`).
  * You must ensure there are no firewall rules that affect the `./well-known/acme-challenge` directory.
  * You must ensure there is no geo-ip blocking
  * LetsEncrypt *does not and will not* publish their outbound IPs
  * The outbound LetsEncrypt IPs are subject to change at any point, and often do change. You should not rely on bypass rules for IPs that you discovered, or that were shared for you - they will eventually break.
  
# For `DNS-01` challenges, LetsEncrypt will connect with the authoritative nameservers for your domain.  The DNS servers **MUST** be on the public internet with no geo-blocking.  If you delegate challenges to a secondary DNS system via CNAMEs, those **MUST** also be on the public internet with no geo-blocking.

If you run geo-blocking on your servers, you have the following options:

* Disable geo-blocking during Certificate procurement. Your firewall may support this via an API, or you can write a script to do this.  The geo-blocking must only be disabled during the certificate procurement process, it can be re-enabled once your ACME client is finished.
* If your setup allows, you may be able to delegate the DNS challenges to a third-party system that does not have geo-blocking.

### Ephemeral Routing Issues

If your ACME Client on a specific machine has successfully obtained Certificates before, but you are suddenly having issues (outbound or inbound) to LetsEncrypt, the leading cause is a temporary routing issue within one of the network providers.  These issues are usually resolved within a few hours, though it can sometimes take a day or two.  This can usually be identified by running a `traceroute` to the LetsEncrypt API and noting a connection being dropped somewhere before the final destination.  Occasionally the routing issue will only be from LetsEncrypt to your server, which can be difficult to detect.

### Issues with SSL Libraries

Some Subscribers have experienced issues due to local SSL installations on their systems.

As of 2024, LetsEncrypt **REQUIRES** TLS 1.2 or better for both inbound API requests and outbound challenge verification connections, [see reference](https://community.letsencrypt.org/t/rejecting-tls-1-0-1-1-for-inbound-acme-connections/176107).  

If you do not meet these requirements, you will experience a connection/socket error.

If your system does not support any of the supported ciphers, you will experience a connection/socket error.

You may also have issues due to how the SSL library is built.  The greater SSL client/library ecosystem adopted an implementation detail several years ago, in which (by default) clients are now allowed to build their own trust paths during chain verification.  This change allowed LetsEncrypt to support a wider range of clients through cross-signs of their roots, however certain openssl builds would have issues.

If you have an older version of OpenSSL (or similar), you may need to update.

### Issues with Root Trust Stores

The LetsEncrypt ACME endpoints are compatible with (LetsEncrypt's own) ISRG X1 Root certificate.  Most active operating systems and libraries support this.

If you have an older system, it must be patched to support this root.

If you have an older system, you may run into issues if the expired `IdentTrust DST Root CA X3` is in your trust store. A quick fix is to remove that from the trust store.  This can happen with legacy machines, such as OSX<=10.14.  The issue is related to previously mentioned change in path-building.

### LetsEncrypt has blocked you

Many people think LetsEncrypt has blocked their IP.  This is almost never the case. 

A rare IP block was discussed in [this thread](https://community.letsencrypt.org/t/obtaining-new-ssl-certificate-on-remote-server/220288/7).

Note the following:

1. A `traceroute` showed connectivity up until the destination IP
2. The ACME Client's error message contains the following error/exception message `TLS/SSL connection has been closed (EOF)`


## Troubleshooting 

LetsEncrypt runs their service on several platforms and networks.  

Currently, the following is known:

* API Requests are fronted by Cloudflare
* OCSP Requests are fronted by Akamai
* Multi-Vantage Challenge Authorizations often come from Amazon networks.

### Troubleshooting tools

#### Traceroute & ping

Try connecting to the ACME endpoint. This can show network issues.

    ping acme-v02.api.letsencrypt.org
    sudo traceroute -T -p443 acme-v02.api.letsencrypt.org

#### Curl

Try connecting via curl or wget

    curl https://acme-v02.api.letsencrypt.org/directory

#### OpenSSL

Is your issue with openssl?

    echo | openssl s_client -connect acme-v02.api.letsencrypt.org:443 | head














