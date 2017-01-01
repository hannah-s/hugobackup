+++
author = "Hannah"
categories = ["letsencrypt", "SSL" ]
date = "2016-12-27T16:15:14+11:00"
description = "It's time to TLS-up, if you haven't switched already"
#featured = "Featured"
#featuredalt = ""
#featuredpath = "featuredpath"
linktitle = "letsencrypt"
title = "ELI5 letsencrypt on CentOS 7.3"
type = "post"

+++

First of all - why HTTPS, or: 'I don't keep credit card numbers or passwords, why should I care about man-in-the-middle attacks or users' privacy?'
<!--more-->

* "There is no such thing as [non-sensitive web traffic](https://https.cio.gov/everything/)"".
* Google's ranking algorithm [penalizes non-TLS secure websites](http://thenextweb.com/google/2015/12/17/unsecured-websites-are-about-to-get-hammered-in-googles-search-ranking/).
* HTTPS is faster! The overhead traffic cost is history - see for yourselves on [httpvshttps.com](http://httpvshttps.com). It's true even if you [ignore](https://www.troyhunt.com/i-wanna-go-fast-https-massive-speed-advantage/) the fact that this test page forces HTTP/2: downgrade to the equally-common HTTP/1.1, and pages still load %19 faster.
* Leading browsers will eventually [warn users](https://www.chromium.org/Home/chromium-security/marking-http-as-non-secure) about unsecure websites in a prominent manner.
* Installation and automatic certificate renewal are super easy with letsencrypt. I dealt with Comodo and GeoTrust's SSL certificate ordering/renewal processes, it wasn't as much of a cakewalk as letsencrypt and Certbot.
* Everyone else is doing it: the number of websites that that force HTTPS has doubled over the last year, according to [Builtwith](https://trends.builtwith.com/ssl/SSL-by-Default).
* It's free!


## Get to work

Apache needs to have SSL enabled and HTTPS-accessible (with a scary security warning) before you start. Else, letsencrypt will return errors on the first run.

Install Apache, start it and ensure it comes back after rebooting:

    [root@centos-vm ~]# yum install httpd mod_ssl -y
    [root@centos-vm ~]# systemctl start httpd
    [root@centos-vm ~]# systemctl enable httpd  

Open port 80 and 443 on firewalld's default zone:

    [root@centos-vm ~]# firewall-cmd  --add-service={http,https} --permanent
    success


Conveniently, the package comes with a premade vhost that's configured for SSL already. This is what it looks like (sans the comments and empty lines), before letsencrypt modifies some stuff:

    [root@centos-vm ~]# egrep -v '#|^$' /etc/httpd/conf.d/ssl.conf
    Listen 443 https
    SSLPassPhraseDialog exec:/usr/libexec/httpd-ssl-pass-dialog
    SSLSessionCache         shmcb:/run/httpd/sslcache(512000)
    SSLSessionCacheTimeout  300
    SSLRandomSeed startup file:/dev/urandom  256
    SSLRandomSeed connect builtin
    SSLCryptoDevice builtin
    <VirtualHost _default_:443>
    ErrorLog logs/ssl_error_log
    TransferLog logs/ssl_access_log
    LogLevel warn
    SSLEngine on
    SSLProtocol all -SSLv2
    SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5:!SEED:!IDEA
    SSLCertificateFile /etc/pki/tls/certs/localhost.crt
    SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
    <Files ~ "\.(cgi|shtml|phtml|php3?)$">
        SSLOptions +StdEnvVars
    </Files>
    <Directory "/var/www/cgi-bin">
        SSLOptions +StdEnvVars
    </Directory>
    BrowserMatch "MSIE [2-5]" \
             nokeepalive ssl-unclean-shutdown \
             downgrade-1.0 force-response-1.0
    CustomLog logs/ssl_request_log \
              "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
    </VirtualHost>  

(Yes, it excludes SSLv2 specifically, but not the [also-vulnerable SSLv3](https://disablessl3.com). We'll worry about it later.)

At this point, there should be a scary browser security warning when attempting to access via HTTPS.

Time to install Certbot. It's not in the default repositories, so we'd want to enable EPEL (Fedora's extra packages) first, and refresh the package list cache.

    [root@centos-vm ~]# yum -y install epel-release
    [root@centos-vm ~]# yum makecache
    [root@centos-vm ~]# yum install -y python-certbot-apache

From now on, it'd be [fairly similar](https://certbot.eff.org/#centosrhel7-apache) to the official guide. Certbot is a TUI - text user interface, and the prompts are rather trivial to follow.

First run, sans some of the prompts:

	[root@centos-vm ~]# certbot --apache --agree-tos -d home.parabiosis.net --uir


It's a good idea to choose the 'secure' option ("--uir"), and disabled non-TLS secure HTTP access altogether (this forwards visitors to HTTPS instead).

## What happened here?

Reading `/etc/httpd/conf.d/ssl.conf` now shows that letsencrypt edited some directives, changing the generic private key, cert and chain paths to the following:

    SSLCertificateFile /etc/letsencrypt/live/home.parabiosis.net/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/home.parabiosis.net/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/home.parabiosis.net/chain.pem

It also added a new vhost, with some mod_rewrite rules to issue a 301 permanent redirect from port 80 to HTTPS:

    [root@centos-vm ~]# egrep -v '#|^$' /etc/httpd/conf.d/le-redirect-home.parabiosis.net.conf
    <VirtualHost _default_:80>
      ServerName home.parabiosis.net
      ServerSignature Off
      RewriteEngine On
      RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,QSA,R=permanent]
      ErrorLog /var/log/httpd/redirect.error.log
      LogLevel warn
    </VirtualHost>

"le-redirect" is probably short for letsencrypt-redirect, but I rather pretend that it's in French and pronounce it accordingly, for fun.


## Testing

Check that the redirect to HTTPS is working:

    [root@centos-vm ~]# curl -I http://home.parabiosis.net/
    HTTP/1.1 301 Moved Permanently
    Date: Sun, 01 Jan 2017 10:35:07 GMT
    Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.1e-fips
    Location: https://home.parabiosis.net/
    Content-Type: text/html; charset=iso-8859-1

If `curl -k` (same as --insecure) works where other curls don't, something is wrong with the SSL configuration.

Check that SSLv3 is disabled [here](https://www.ssllabs.com/ssltest/analyze.html?d=home.parabiosis.net), or using this:

    [root@centos-vm ~]# openssl s_client -connect home.parabiosis.net:443 -ssl3

If the above returns that renegotiation was successful - you're [vulnerable](https://disablessl3.com) to POODLE attacks. Modify the ssl.conf vhost from earlier to exclude it.

    [root@centos-vm ~]# grep SSLProtocol /etc/httpd/conf.d/ssl.conf
    SSLProtocol all -SSLv2 -SSLv3
    [root@centos-vm ~]# systemctl reload httpd

This is the sought after result - see "handshake failure":

    [root@centos-vm ~]# openssl s_client -connect home.parabiosis.net:443 -ssl3
    CONNECTED(00000003)
    139755703465888:error:14094410:SSL routines:SSL3_READ_BYTES:sslv3 alert handshake failure:s3_pkt.c:1259:SSL alert number 40
    139755703465888:error:1409E0E5:SSL routines:SSL3_WRITE_BYTES:ssl handshake failure:s3_pkt.c:598:

## Renewals with systemd timers

There's no risk of too frequent renewals (the cap is pup to 5 renewals a day), because certbot doesn't renew the cert unless its time is up:

        [root@centos-vm ~]# certbot renew --verbose
        Root logging level set at 10
        Saving debug log to /var/log/letsencrypt/letsencrypt.log
        certbot version: 0.9.3
        Arguments: ['--verbose']
        Discovered plugins:   PluginsRegistry(PluginEntryPoint#apache,PluginEntryPoint#webroot,
        PluginEntryPoint#null,PluginEntryPoint#manual,PluginEntryPoint#standalone)

        -------------------------------------------------------------------------------
        Processing /etc/letsencrypt/renewal/home.parabiosis.net.conf
        -------------------------------------------------------------------------------
        Cert not yet due for renewal

        The following certs are not due for renewal yet:
        /etc/letsencrypt/live/home.parabiosis.net/fullchain.pem (skipped)
        No renewals were attempted.
        no renewal failures

Add systemd timer (the new cron-job-like feature):

        [root@centos-vm ~]# touch /etc/systemd/system/letsencrypt.{service,timer}


Edit the letsencrypt.service file to look like this:

        [root@centos-vm ~]# cat /etc/systemd/system/letsencrypt.service
        [Unit]
        Description=letsencryptn renewal

        [Service]
        Type=oneshot
        ExecStart=/usr/bin/certbot renew --quiet

Edit the letsencrypt.timer file to look like this:

        [root@centos-vm ~]# cat /etc/systemd/system/letsencrypt.timer
        [Unit]
        Description=letsencrypt renewals

        [Timer]
        OnCalendar=hourly
        Unit=letsencrypt.service

        [Install]
        WantedBy=multi-user.target

Start and enable for next reboots:

        [root@centos-vm ~]# systemctl start letsencrypt.timer
        [root@centos-vm ~]# systemctl enable letsencrypt.timer


Yup, it's scheduled:

        [root@centos-vm ~]# systemctl list-timers letsencrypt*
        NEXT                          LEFT       LAST PASSED UNIT              ACTIVATES
        Sun 2017-01-01 23:00:00 AEDT  38min left n/a  n/a    letsencrypt.timer letsencrypt.service

        1 timers listed.
        Pass --all to see loaded but inactive timers, too.

## Possible pitfalls

1) Turns out, my ISP blocks by default a series of ports, to any address that isn't within their network, such as 25, 80, 443, 139, 135 and a couple more that I'm forgetting. While this allowed me HTTP/HTTPS access from my own network, even though external interfaces, it prevented remote connections from letsencrypt CA. This feature needed to be disabled on their web UI.

2) On my Ubuntu Xerus 16.04, `apt-get install apache2` installs mod_ssl by itself. However, it doesn't enable it. I couldn't figure out why http://site.com:443 loads, but https://site.com doesn't. Some head-breakage later, enabling the mod (adds a symlink to an included dir) helped:

		hannah@lolpc:~/parabiosis.net$ sudo a2enmod ssl
		Considering dependency setenvif for ssl:
		Module setenvif already enabled
		Considering dependency mime for ssl:
		Module mime already enabled
		Considering dependency socache_shmcb for ssl:
		Enabling module socache_shmcb.
		Enabling module ssl.
		See /usr/share/doc/apache2/README.Debian.gz on how to configure SSL and create 	self-signed certificates.
		To activate the new configuration, you need to run:
  	service apache2 restart

On CentOS 7.3, no need to enable - the default Apache configuration already has `/etc/httpd/conf.modules.d/00-ssl.conf` covered:

	[root@centos-vm ~]# grep Include /etc/httpd/conf/httpd.conf
	# LoadModule foo_module modules/mod_foo.so
	Include conf.modules.d/`*`.conf


## Alternatives

This page has an SSL cert issued by Amazon's CA, thanks to the fact that they started offering them too, free of charge, since last January (wildcards too!). On AWS, you pay for the application usage, not the cert. Being that they also take care of the renewals automatically, it's just as good and easy a solution [(guide)](https://medium.com/@arcdigital/enabling-ssl-via-aws-certificate-manager-on-elastic-beanstalk-b953571ef4f8#.mm3tlftt6).

[There are solutions](https://nparry.com/2015/11/14/letsencrypt-cloudfront-s3.html) out there for using letsencrypt on AWS, but in order to make it work, you can't enjoy the freebie ALIAS record on Route53, and you'd have to use paid A and/or AAAA records.

If you must, [Comodo offer a 90-days cert too](https://ssl.comodo.com/free-ssl-certificate.php), but the renewal won't be as nicely automated.
