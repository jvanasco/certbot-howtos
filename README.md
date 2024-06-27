This repository is a collection of HowTos and utilities for dealing with SSL certificates and similar things.

I developed a proprietary ACME Client and SSL Certificate Manager that was open sourced as [Peter_SSLers](https://github.com/aptise/peter_sslers), along with an OpenResty package designed to dynamically load SSL Certificates into Nginx servers on demand [lua-resty-peter_sslers](https://github.com/aptise/lua-resty-peter_sslers).

During that development, I learned a lot from the LetsEncrypt Community and ISRG staff.  I was kindly elevated to "Community Leader" status on the [LetsEncrypt Community](https://community.letsencrypt.org/u/jvanasco/summary) website.  My main interests there are RFC/API/Troubleshooting improvements, scalable/distributed ACME clients, and other complex integrations (i.e. distributed devices).  I've contributed code and docs to multiple ACME servers and clients.

# Python Tools

* **cert_utils** Python toolkit for generic SSL Certificate functions. Attempts to use pure-Python, with a failover to OpenSSL via subproceses.
  * [github src](https://github.com/aptise/cert_utils)
  * [pypi](https://pypi.org/project/cert-utils/)

* **Peter_SSLers** Integrated ACME Client and Certificate Manager
  * [github src](https://github.com/aptise/peter_sslers)

* **lua-resty-peter_sslers** OpenResty integration for dynamic certificate loading via SNI
  * [github src](https://github.com/aptise/lua-resty-peter_sslers)


* **fake_server.py** Used to test proxypass integrations for HTTP-01 ACME challenges
  * [github src](https://github.com/aptise/peter_sslers/blob/main/tools/fake_server.py)

* **zone_updates.py** CloudFlare API script to migrate an IP address across your account (for when a server ip changes), and enable Certificate Transparency monitoring
  * [github gist](https://gist.github.com/jvanasco/5b459923d887ec8dfde51a0d529a2242)
  

* **replace_domain.py** Script to manipulate an acme-dns database to utilize human-oriented FQDNs instead of UUIDs
  * [github src](https://github.com/aptise/peter_sslers/blob/main/tools/replace_domain.py)

# Certbot HOWTOs

These howto files describe my preferred ways to integrate Certbot.

* [acme-dns](certbot--acme-dns.md)
* [nginx](certbot--nginx.md)

# LetsEncrypt HOWTOs

* [troubleshoot-connectivity](letsencrypt--troubleshoot-connectivity.md)


# Third Party Tools

* [LetsDebug](https://letsdebug.net/) Online tool that can troubleshoot miscellaneous issues with LetsEncrypt issuance
* [UnboundTest](https://unboundtest.com/) Online tool that mimics LetsEncrypt's DNS resolver








