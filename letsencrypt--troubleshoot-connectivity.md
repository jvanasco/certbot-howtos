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
  
* For `DNS-01` challenges, LetsEncrypt will connect with the authoritative nameservers for your domain.  The DNS servers **MUST** be on the public internet with no geo-blocking.  If you delegate challenges to a secondary DNS system via CNAMEs, those **MUST** also be on the public internet with no geo-blocking.

If you run geo-blocking on your servers, you have the following options:

* Disable geo-blocking during Certificate procurement. Your firewall may support this via an API, or you can write a script to do this.  The geo-blocking must only be disabled during the certificate procurement process, it can be re-enabled once your ACME client is finished.
* If your setup allows, you may be able to delegate the DNS challenges to a third-party system that does not have geo-blocking.


#### Typical Errors

* > Hint: The Certificate Authority failed to download the temporary challenge files created by Certbot. Ensure that the listed domains serve their content from the provided --webroot-path/-w and that files created there can be downloaded from the internet.

* > certbot.errors.AuthorizationError: Some challenges have failed.



### Ephemeral Routing Issues

If your ACME Client on a specific machine has successfully obtained Certificates before, but you are suddenly having issues (outbound or inbound) to LetsEncrypt, the leading cause is a temporary routing issue within one of the network providers.  These issues are usually resolved within a few hours, though it can sometimes take a day or two.  This can usually be identified by running a `traceroute` to the LetsEncrypt API and noting a connection being dropped somewhere before the final destination.  Occasionally the routing issue will only be from LetsEncrypt to your server, which can be difficult to detect.

#### Typical Errors

You may see errors like these:

* > ReadTimeout: HTTPSConnectionPool(host=‘acme-v01.api.letsencrypt.org 1’, port=443): Read timed out. (read timeout=45)

* > requests.exceptions.ConnectionError: HTTPSConnectionPool(host='acme-v02.api.letsencrypt.org', port=443): Max retries exceeded with url: /directory (Caused by NewConnectionError('<urllib3.connection.VerifiedHTTPSConnection object at 0x7f55b6263f60>: Failed to establish a new connection: [Errno 101] Network is unreachable',))

* > ValueError: Requesting acme-v02.api.letsencrypt.org/directory: Network is unreachable
Please see the logfiles in /var/log/letsencrypt for more details.




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

#### Typical Errors

You may see errors like these:

*  requests.exceptions.SSLError: HTTPSConnectionPool(host='acme-v02.api.letsencrypt.org', port=443): Max retries exceeded with url: /directory (Caused by SSLError(SSLError("bad handshake: Error([('SSL routines', 'tls_process_server_certificate', 'certificate verify failed')])")))


### Date/Time Issue

Sometimes a server has the incorrect date/time set.  Sometimes a computer's battery dies and the date is reset.

When these things happen, it is likely to encounter Certificates that are not yet valid - which will cause API operations to fail.

Fixing the system time should resolve this.

#### Typical Errors

* > File "/usr/lib/python3/dist-packages/urllib3/util/retry.py", line 592, in increment
raise MaxRetryError(_pool, url, error or ResponseError(cause))
urllib3.exceptions.MaxRetryError: HTTPSConnectionPool(host='acme-v02.api.letsencrypt.org', port=443): Max retries exceeded with url: /directory (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: certificate is not yet valid (_ssl.c:992)')))


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


## Edge Cases


### PyOpenSSL Error - Rust Bindings

Via https://community.letsencrypt.org/t/did-openssl-3-0-break-certbot/207661

A user of FreeBSD reported a Python exception in Certbot that ended with:

>    File "/usr/local/lib/python3.9/site-packages/cryptography/exceptions.py", line 9, in <module>
>      from cryptography.hazmat.bindings._rust import exceptions as rust_exceptions
> ImportError: /usr/local/lib/python3.9/site-packages/cryptography/hazmat/bindings/_rust.abi3.so: Undefined symbol "EVP_default_properties_is_fips_enabled"

The issue was twofold:

    * FreeBSD had a bug in their openssl library linking
    * A Python module used by Certbot (cryptography) migrated a FIPs check to use rust, which caused a fatal exception
    
The long fix is waiting for FreeBSD to fix their issue.  See https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=273770

The short fix is to downgrade the cryptography *in the relevant Python environment*:

    pip install "cryptography==40.0.2"













