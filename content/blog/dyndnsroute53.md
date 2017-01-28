+++
author = "Hannah"
categories = ["DNS", "AWS", "Route53" ]
date = "2017-01-12T01:59:12+11:00"
description = "I wrote up a thing to automate updating my dynamic IP."
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Dynamic DNS with Route53"
type = "post"

+++

Once upon a time, I found an old D-Link 514 modem-router, on hard rubbish collection day. It was of no real use to me, but [dlinkddns.com](http://dlinkddns.com) used to let you sign up for a free dynamic DNS service if you provided a serial number of an actual D-Link device you owned. I had some subdomain.dlinkddns.com registered, with a CNAME from a more elegant looking domain set to point at it. My modem had an integrated feature to push updates to most popular DNS providers. All was well with the world. 

I used it for a while, and then I didn't. Eventually I lost the login details, and I haven't the foggiest idea what they were. I know the passphrase was a line from  a song, but the band in subject has too extensive a discography to remember what it was. When I bumped into [awscli's Route 53 command reference](http://docs.aws.amazon.com/cli/latest/reference/route53/), it started to sound like a nice low-cost alternative to no-ip, dyndns etc.

I wrote up a quick [DNS-updater-cronjob-thingy](https://github.com/hannah-s/dynamicdns/). There's probably easier/faster ways to do it, but it's pretty basic and easy to read.

Clone it and stuff:

	git clone https://github.com/hannah-s/dynamicdns.git ~/dynamicdns

Install a cron job (doesn't require root level access) with 'crontab -e' and pasting:

	*/5 * * * *  /usr/bin/flock -n '/tmp/dynamicdns.lock' ~/dynamicdns/aws-dyndns.sh | logger -t aws-dynamic-dns

List it:

	hannah@lolpc:~$ crontab -l
	*/5 * * * *  /usr/bin/flock -n '/tmp/dynamicdns.lock' ~/dynamicdns/aws-dyndns.sh | logger -t aws-dynamic-dns


'Flock' will use a lock file prevent a billion stuck cron jobs to build up, should something go wrong. 

The logger addition in the end makes it a bit easier for me to sift through syslog entries, by adding a prefix:

	hannah@lolpc:~$ tail -f /var/log/syslog | grep aws-dynamic-dns
	Jan 12 02:40:03 lolpc aws-dynamic-dns: Current DNS record: 202.161.17.103.
	Jan 12 02:40:03 lolpc aws-dynamic-dns: The external IP address has changed to 124.149.154.188.
	Jan 12 02:40:03 lolpc aws-dynamic-dns: Updating the A record for whatever.domain.com.
	Jan 12 02:40:04 lolpc aws-dynamic-dns: {
	Jan 12 02:40:04 lolpc aws-dynamic-dns:     "ChangeInfo": {
	Jan 12 02:40:04 lolpc aws-dynamic-dns:         "Id": "/change/XXXXXXXX",
	Jan 12 02:40:04 lolpc aws-dynamic-dns:         "SubmittedAt": "2017-01-11T15:40:04.553Z",
	Jan 12 02:40:04 lolpc aws-dynamic-dns:         "Comment": "Update DNS record for whatever.domain.com at 2017-01-12:02:40:03.",
	Jan 12 02:40:04 lolpc aws-dynamic-dns:         "Status": "PENDING"
	Jan 12 02:40:04 lolpc aws-dynamic-dns:     }
	Jan 12 02:40:04 lolpc aws-dynamic-dns: }
	Jan 12 02:40:04 lolpc aws-dynamic-dns: Done.

My pal Joel, who's a much better tech than me, suggested to add a 'mail -s' sort of line in order to send myself an email update whenever the IP changes. I decided against it eventually, because it requires installing an MTA, i.e. Postfix, and configuring it to use my ISP's MTA as a smarthost, and then applying various security measures against spammers. If you don't mine the effort, and/or have an mailutils installed and configured already (`which mail` returns an error typically means nope), it's as simple as adding this after the "Done" line:

	mail -s "Notice: your IP has changed." "example@example.com" <<EOF
	The new IP is $NEWIP.
	EOF



