# HOWTO - Certbot + AcmeDNS

This is a quick guide covering my preferred integration of [Certbot](https://github.com/certbot/certbot) and [acme-dns](https://github.com/joohoi/acme-dns).

## Why acme-dns?

In certain situations, one *must* utilize `DNS-01` challenges to pass ACME Authorizations. Examples include: wildcard certificates, split-horizon DNS, and non-public DNS.

While many DNS providers offer API access that can be used to automate certificate procurement, many lack the ability to limit what an API Token can do. This creates significant security vulnerabilities, as the same token used to automate DNS-01 challenges might also be able to remove the A records for the domain or even transfer ownership of a domain to a malicious actor.  acme-dns was designed to eliminate these concerns and does a phenominal job.  For more information, you can read this article by the project's maintainer: [A Technical Deep Dive: Securing the Automation of ACME DNS Challenge Validation](https://www.eff.org/deeplinks/2018/02/technical-deep-dive-securing-automation-acme-dns-challenge-validation)

## General Overview

This setup has a few parts that may make it seem complex and intimidating, but it's quite simple.

1- Operate an acme-dns server on the public internet
2- utilize firewall rules to enable/disable acme-dns as needed
3- utilize an alternate configuration of Certbot for local needs


## Install and Configure acme-dns on the PUBLIC server

acme-dns **MUST** be installed on a server connected to the public internet.

You **MUST** own a domain name to operate acme-dns on.

The acme-dns docs use `auth.example.org`; I prefer to use `acme-dns.example.com`.
You can use any registerable domain on the public internet.
This domain does not need any relation to the domains that will use acme-dns for authorization.
This domain will function just like a cloud vendor or service provider.

The installation and core configuration of acme-dns is outlined on the project's homepage.  Follow these instructions:

https://github.com/joohoi/acme-dns

Some users have been confused with the DNS requirements for the system. These lines from the documentation are the most important (slightly edited):

> You will need to add 2-3 DNS records on your domain's regular DNS server:
> 
> `NS` record for `auth.example.com` pointing to `auth.example.com` (this means, that `auth.example.com` is responsible for any `*.auth.example.com` records)
> `A` record for `auth.example.com` pointing to the IPv4-ADDRESS of the server
> If using IPv6, an `AAAA` record pointing to the IPv6 address.

The acme-dns configuration is fairly straightforward.  Follow the docs.  Test that it is working!

Pay attention to the section about installing it into systemd for service management.  


## Configure IP Tables on the PUBLIC server

This step is not required, but many people have Security concerns about leaving unnecessary ports open on public systems.

acme-dns requires 3 ports to be open for use: port 53 UDP and 53 TCP to handle DNS queries, and 8011 (configurable) for the HTTPS API.

To easily enable/disable these on demand, we will create an IP Tables Chain.

This allows us to only run acme-dns as needed:

* create the acme-dns chain

	iptables -N acme-dns

* set the tables to run it BEFORE the port 53 deny

    ```
	iptables -A INPUT -j acme-dns
    ```

* it make make sense to first do a backup...

    ```
	iptables-save > iptables.dump
	vi iptables.dump
    ```

* then manually add in the acme-dns line as the first chain. It might look something like this:

    ```
	:INPUT DROP [0:0]
	:FORWARD ACCEPT [0:0]
	:OUTPUT ACCEPT [451:186415]
	:acme-dns - [0:0]
	:f2b-postfix - [0:0]
	:f2b-sshd - [0:0]
	:f2b-sshd-invalid_accounts - [0:0]
	:f2b-sshd-system_accounts - [0:0]
	-A INPUT -j acme-dns
    ```

* then reload the edited dumpfile, so it persists across reboots

    ```
	iptables-restore < iptables.dump
    ```
	

The result of this work, is you will be able to enable acme-dns with the following:


    ```
	iptables -A acme-dns -p tcp --dport 53 -j ACCEPT
	iptables -A acme-dns -p udp --dport 53 -j ACCEPT
	iptables -A acme-dns -p tcp --dport 8011 -j ACCEPT
    ```

For those unfamiliar with iptables, we're just adding telling it to add those ACCEPT rules into the chain.

and then you can disable it with the following:

    ```
	iptables -F acme-dns
    ```

Again, for those unfamiliar with iptables, we're just flushing all the rules in our acme-dns chain.

Once this is running, use the testing information (`curl` instructions) from the acme-dns project to test if your firewall rules work as intended.  Make sure you can not connect when the rules are flushed, and that you can connect when the rules are loaded

## Install Certbot (or other ACME Client)

### Notes on ACME Clients

For this HOWTO, we'll be using Certbot as the ACME client, then integrating it with our acme-dns install via a plugin.

Certbot is not required to use acme-dns.  Several well-made ACME clients have built-in acme-dns support.

There are 4 general types of ACME clients:

* Lightweight clients that only procure Certificates
* Midrange clients that procure Certificates and can configure web servers (Certbot)
* Enterprise clients capable of managing large installations
* clients built into web servers or certificate managers

We are using Certbot in this howto for several reasons:

* Easy to install/manage/configure
* acme-dns plugins are available
* Easy to use on the server or a workstation
* Certbot supports multiple hooks that simplify work
* Certbot supports customizing where you save files

### Install Certbot(s) for Server Use

Currently, the recommended way to install Certbot is via the Snap distribution system.

Users **should not** install Certbot from platform packages. They are often old and lack important improvements.

While the Snap distribution system has many advantages, a significant disadvantage is that it generally requires plugins to be installed from the Snap system as well.  While Certbot packages and publishes their official plugins for the Snap ecosystem, most third-party developers do not.

While it is possible to leverage system Python packages into a snap installation, the process can be complex.

My recommendation is to install via PIP for server use.

Your best option is to simply follow the instructions on https://certbot.eff.org for PIP installation.

### Install Certbot(s) with PIP for Local Workstation Use

The following is how I do a LOCAL installation on my workstation.

I use a Mac, and prefer to install Certbot (and other projects) the following way:

* my user directory has a `~/dev-env` directory for virtual environments. Certbot will have a dedicated venv here.
* I have a dedicated `/Development` volume for projects that has different backup/archiving rules. Certbot will save files here.

Some online tutorials configure Certbot to save files into the virtualenv.  IMHO this is a bad idea and should be avoided.

The reasons for managing two directories of files:

* A new virtualenv will needed for each Python version; you may need to install a newer Python version onto your system to install a more recent Certbot
* Decoupling the datastore means different Certbots/Python environments can access a centralized datastore
* We're going to invoke Certbot to save files into specific "userland" places, so they can be part of our data backups, instead of system directories. 

I typically have multiple versions of Python installed, so my virtualenv will include the Python version in it.

The following **assumes** `python` is mapped to the latest Python 3.  You may need to invoke `python3` or a specfic ``/path/to/python`

* What version of Python do you have?

    ```
    python --version
    ```
   
* What version of pip do you have?  is it for the right Python?

    ```
    pip --version
    which pip 
    ```
   
* Once you have the right pip, let's update!

    ```
    sudo pip install --upgrade pip virtualenv
    ```
   
* Create the virtualenv. If you have another version of Python, replace the "3.12" with your version number:

    ```
    mkdir ~/dev-env
    cd ~/dev-env
    python3 -m venv certbot-3.12
    ```

* activate the environment.  We'll still use absolute commands, but this is a good thing to do:

    ```
    source ~/dev-env/certbot-3.12/bin/activate
    ```

* install Certbot:

    ```
    ~/dev-env/certbot-3.12/bin/pip install certbot
    ```

* check to see you have the right Certbot...

    ```
    which certbot
    ```
    
* you should see something like:

    ```
    /Users/jvanasco/dev-env/certbot-3.12/bin/certbot
    ```
    
* Finally, we're going to create our centralized datastore. The root directory can be anywhere on your system, but will need subdirectories for: `-logs`, `-work`, `authorization_hooks`, and `etc/letsencrypt`.

    ```
    mkdir /Volumes/Development/tools/certbot-local
    mkdir /Volumes/Development/tools/certbot-local/-logs
    mkdir /Volumes/Development/tools/certbot-local/-work
    mkdir /Volumes/Development/tools/certbot-local/authorization_hooks
    mkdir /Volumes/Development/tools/certbot-local/etc
    mkdir /Volumes/Development/tools/certbot-local/etc/letsencrypt
    ```

## Install Authentication Hook

acme-dns requires an authentication hook when using Certbot.

Several are listed on the acme-dns project page.

My preferred hook is the Python hook [acme-dns-certbot-joohoi](https://github.com/joohoi/acme-dns-certbot-joohoi)

We are basically going to follow the instructions on the page, but with a slightly different install location - the root of our centralized datastore:

    ```
    cd /Volumes/Development/tools/certbot-local/authorization_hooks
    curl -o https://raw.githubusercontent.com/joohoi/acme-dns-certbot-joohoi/master/acme-dns-auth.py
    chmod 0700 acme-dns-auth.py
    ```

Then you should edit the configuration values as instructed on that project's docs.

The important things to note in this edit - the storage path should be in your centralized datastore:

    ```
    # Path for acme-dns credential storage
    STORAGE_PATH = "/Volumes/Development/tools/certbot-local/authorization_hooks/acmedns.json"
    ```
    

## Onboarding a Certificate

### Local Usage

In this example, we're going to onboard a wildcard certificate for `example.com` into our system.

To do this, we're going to need two terminal windows:

* Terminal A: ssh into your public internet server
* Terminal B: stays on your local machine

We will also need to edit the main DNS servers for `example.com`. This only needs to be done once.

So here we go:

#### Terminal A - public internet server

In the first window, we will ssh into our public server and "switch on" the acme-dns system:

    ```
    ssh acme-dns.example.com
    sudo bash
    iptables -A acme-dns -p tcp --dport 53 -j ACCEPT
    iptables -A acme-dns -p udp --dport 53 -j ACCEPT
    iptables -A acme-dns -p tcp --dport 8011 -j ACCEPT
    systemctl start acme-dns.service    
    ```

To automate this, you might want to wrap the `iptables` and `systemctl` lines into an executable script.

Once acme-dns is enabled, we're going to need a terminal window on the local machine:

#### Terminal B - local

The first thing to do is activate our virtualenv:

    ```
    source ~/dev-env/certbot-3.12/bin/activate
    which certbot
    ```

Once that is done, while we *can* use the bare `certbot` command as we know it is properly aliased, it is still recommended to specify the exact executable.

Here is the command we invoke.  I will describe it in detail below:

    ```
    ~/dev-env/certbot-3.12/bin/certbot \
        --config-dir /Volumes/Development/tools/certbot-local/etc/letsencrypt \
        --work-dir /Volumes/Development/tools/certbot-local/-work \
        --logs-dir /Volumes/Development/tools/certbot-local/-logs \
        certonly \
        --manual \
        --manual-auth-hook "/Volumes/Development/tools/certbot-local/acme-dns-auth.py" \
        --preferred-challenges dns \
        --debug-challenges \
        --server https://acme-staging-v02.api.letsencrypt.org/directory \
        -d example.com \
        -d *.example.com
    ```

The command flags for `--config-dir`, `--work-dir` and `--logs-dir` are just telling Certbot to operate that functionality out of our centralized datastore.  This is important for managing these files, and troubleshooting.

The `certonly` command means that we are not trying to configure a webserver -- Certbot is just functioning as an ACME client.

The `--manual` command is what lets us integrate with acme-dns.  The `--manual-auth-hook` points to our hook script.

The `--preferred-challenges dns` tells Certbot that we want to utilize `DNS-01`.

The `--debug-challenges` flag is required to pause Certbot so you can edit the DNS records.

When this command is invoked, there will be quite a bit of output before Certbot prints a message that looks like the following:

```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Challenges loaded. Press continue to submit to CA.
The following FQDNs should return a TXT resource record with the value
mentioned:
FQDN: _acme-challenge.example.com
Expected value: {{EXPECTED_VALUE}}
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Your should *ignore this message* and scroll up.  Several lines above you should see a message that reads:

```
Hook '--manual-auth-hook' for example.com ran with output:
 Please add the following CNAME record to your main DNS zone:
 _acme-challenge.example.com CNAME aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee.acme-dns.example.com.
```

Your main DNS must be configured with that CNAME record (the next step)

#### Update your main DNS with the CNAME record

You might do this on a web control panel, or via a commandline api client.

This will only be done once.

Behind the scenes, the authorization hook we installed had negotiated a new account on the acme-dns server and configured everything to work.

We just need to delegate the DNS-01 challenge from the main dns to our acme-dns installation.

This is done by creating a CNAME record from the main dns onto a unique subdomain that acme-dns has created for this domain.

This CNAME must remain in-place for renewals to happen. If this CNAME is removed, you must either recover it or repeat the onboarding process.

While TXT challenges can be "cleaned up", those exist on the acme-dns server.  This record is the delegation that glues the integration together so it must persist.

Set this record, then test it:

    ```
    dig _acme-challenge.example.com TXT
    ```
    
The results should show something like:

    ```
    ;; ANSWER SECTION:
    _acme-challenge.example.com. 120 IN CNAME aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee.acme-dns.example.com.
    aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee.acme-dns.aptise.com. 1 IN TXT "A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A1A"
    aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee.acme-dns.aptise.com. 1 IN TXT "B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B2B"
    ```
    
If you don't get a success, wait a few moments and try again.  The DNS records need to propagate.

Once you do have success, go back to your local terminal:

#### Terminal B - local

Now that your CNAME record and challenges are known to work, we can follow the Certbot instructions:

    ```
    Challenges loaded. Press continue to submit to CA.
    ```
    
Just hit enter/return.  Your Certificate should quickly issue, be saved to your datastore, and Certbot will exit.

You can continue onboarding domains with additional `certbot` commands.  When done, we can turn acme-dns off.


#### Terminal A - public server

Now that Certbot is done, back in the other window, we just shut acme-dns off:

    ```
    iptables -F acme-dns
    systemctl stop acme-dns.service
    ```
    

## Renewing a Certificate

### Local Usage

Renewing a certificate is incredibly fast and easy.  It requires no interaction, as everything was set up for automation during onboarding.

The abridged version:

#### Terminal A - public internet server

Just like onboarding, ssh into our public server and "switch on" the acme-dns system:

    ```
    ssh acme-dns.example.com
    sudo bash
    iptables -A acme-dns -p tcp --dport 53 -j ACCEPT
    iptables -A acme-dns -p udp --dport 53 -j ACCEPT
    iptables -A acme-dns -p tcp --dport 8011 -j ACCEPT
    systemctl start acme-dns.service    
    ```

#### Terminal B - local

Activate our virtualenv:

    ```
    source ~/dev-env/certbot-3.12/bin/activate
    which certbot
    ```

Then renew your certificates:

    ```
    certbot \
        --config-dir /Volumes/Development/tools/certbot-local/etc/letsencrypt \
        --work-dir /Volumes/Development/tools/certbot-local/-work \
        --logs-dir /Volumes/Development/tools/certbot-local/-logs \
        renew 
    ```

And when you're done...

#### Terminal A - public internet server

Shut off acme-dns

    ```
    iptables -F acme-dns
    systemctl stop acme-dns.service
    ```
    
## Making things even easier

Certbot offers many hooks you can use to automate this work.

Here are some that I like:

* `--deploy-hook` only runs after a *successful* certificate procurement, and will populate the environment with several variables that can be used to automate the certificate distribution.  I often use this hook to do things like:
  * enroll the certificate into a secure version control system that encrypts data. I like to use [Blackbox](https://github.com/StackExchange/blackbox) to encrypt this data for normal Git repositories.
  * deploy the certificates to remote servers. a simple shell script can copy these files with `rsync`, however the remote servers will need to be configured to restart when they detect changes or periodically. an alternative is to use [Fabric](https://fabfile.org) to write a quick script that deploys the certificate and ssh's into the server to restart the affected services.

* `--pre-hook` and `--post-hook` can be used to invoke a script that ssh's into the remote server to enable/disable the firewall rules and acme-dns.  A caveat is these run for each renewal attempt.  While they do not run for certificates that do not need renewal, if you have a large integration this can waste time.  
  * I prefer to handle renewals with a Fabric command that does the following:
    * enable acme-dns and firewall on the remote server
    * invoke certbot locally with the requisite commandline flags and hooks
    * disable acme-dns and firewall on the remote server

    The caveat to this approach is the remote system will be enabled for a few seconds, even if no certificates are needed.  I prefer that to the alternative.












