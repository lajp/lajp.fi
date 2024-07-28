---
title: How I setup my email server
description: A walkthrough of how my mail server is setup. It utilizes Postfix and Dovecot and features spam-filtering, SNI based TLS and support for multiple domain names
published_date: 2022-07-07 11:42:16.717424828 +0000
layout: default.liquid
tags: ["linux"]
is_draft: false
---
# Setting up an email server

## Introduction

In this article I go over my email server setup.

I also attempt to provide a step-by-step guide for the reader to follow and to potentially setup their own server with

In this tutorial I will be setting up an email server for the domains **example1.com** and **example2.com**.
The person following this tutorial can fill in their own domain(s) and add more domains. This setup should work with as many domains as one pleases

Although all the domains will work fine, it is mandatory to choose a primary domain out of them and use that one when for example prompted for **/etc/mailname**

In this tutorial I use a ubuntu-based system. Some steps and configurations may vary from distribution to distribution.

It is important that you follow and read through the tutorial with though and don't just copy the commands into the terminal. :)

## Basic DNS

Getting an email server begins first with acquiring a domain and soon thereafter configuring the DNS for that domain

There are two (or three if you want IPv6) records you need to configure in this initial step:

* `A mail.example1.com <ipv4_of_your_server>`
* `AAAA mail.example1.com <ipv6_of_your_server>`
* `MX example1.com mail.example1.com`

Configure the above records for all of your domains

## Setting up SSL

In this tutorial I'm using certbot to easily setup SSL.

Just run the following commands:

```sh
apt-get update && apt-get upgrade -y
apt-get install certbot
certbot certonly --standalone -d mail.example1.com -d mail.example2.com
```

Your certificates should now be generated and you should be ready to move to the next step

## Installing the dependencies

In this step we install all the programs used in the setup

If you don't plan on using a component of this setup, here's your chance to not install it in the first place.

When running these commands dpkg will prompt you to configure postfix. On the first screen select "Internet Site" and when prompted for the "System mail name" enter your primary domain which in this case is **example1.com**

```sh
apt-get install postfix dovecot-imapd dovecot-sieve spamassassin spamc opendkim opendkim-tools
```

A brief breakthrough of the individual packages and what they are used for

* **postfix**: The legendary <span class="definition" title="mail transfer agent">MTA</span> is one of the two main components of this setup.
    Postfix is responsible for sending/receiving mail to/from millions of other mail server around the world through the
    <span class="definition" title="simple mail transfer protocol">SMTP</span>.
* **dovecot (-imapd, -sieve)**: Dovecot is the other main component.
    It handles the imap server along with locally delivering the mail to the mailboxes and authentication
    (postfix also uses dovecot for authentication).
    The sieve and imapd are optional components for dovecot.
    The sieve does local mailbox management and mail delivery while the imapd handles requests made through the
    <span class="definition" title="internet mail access protocol">IMAP</span>
* **Spamassassin, spamc**: Spamassassin does spam filtering of the mail.
* **opendkim (, -tools)**: DKIM is a widely adopted internet standard that is used to prevent email spoofing.
    It is also particularly useful in making ones mails seem more legitimate and therefore in getting our mail through the spam filters of others.
    We use opendkim for signing our own sent mail and for verifying the signatures of mail sent to us by others.

## Configuring dovecot

We start our configuration from dovecot which I generally consider to be easier than configuring postfix.

### Add stuff to config file

For dovecot the configuration is as simple as copying the following stuff to **/etc/dovecot/dovecot.conf** and changing/adding in your own domain(s)

```
disable_plaintext_auth = no
mail_privileged_group = mail

auth_mechanisms = plain login
auth_username_format = %n

protocols = $protocols imap

mail_location = maildir:~/Mail:INBOX=~/Mail/Inbox:LAYOUT=fs
namespace inbox {
    inbox = yes

    mailbox Drafts {
        special_use = \Drafts
        auto = subscribe
    }

    mailbox Junk {
        special_use = \Junk
        auto = subscribe
    }

    mailbox Sent {
        special_use = \Sent
        auto = subscribe
    }

    mailbox Trash {
        special_use = \Trash
        auto = subscribe
    }

    mailbox Archive {
        special_use = \Archive
        auto = subscribe
    }
}

userdb {
    driver = passwd
}

passdb {
    driver = pam
}

protocol lda {
    mail_plugins = $mail_plugins sieve
}

protocol lmtp {
    mail_plugins = $mail_plugins sieve
}

plugin {
    sieve = ~/.dovecot.sieve
    sieve_default = /var/lib/dovecot/sieve/default.sieve
    sieve_dir = ~/.sieve
    sieve_global_dir = /var/lib/dovecot/sieve/
}


service auth {
    unix_listener /var/spool/postfix/private/auth {
        mode = 0660
        user = postfix
        group = postfix
    }
}

ssl = required
ssl_cert = </etc/letsencrypt/live/mail.example1.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example1.com/privkey.pem

local_name mail.example1.com {
    ssl_cert = </etc/letsencrypt/live/mail.example1.com/fullchain.pem
    ssl_key = </etc/letsencrypt/live/mail.exmaple1.com/privkey.pem
}

local_name mail.example2.com {
    ssl_cert = </etc/letsencrypt/live/mail.example2.com/fullchain.pem
    ssl_key = </etc/letsencrypt/live/mail.example2.com/privkey.pem
}
```

### Restart dovecot

After a quick restart of the dovecot service and everything should be done on that part

```sh
systemctl restart dovecot
```

## Initial postfix configuration

Configuring postfix is a bit trickier

Luckily for us, debian/ubuntu provides a sane default configuration where we can just add/fill in some additional stuff

### Edit stuff in /etc/postfix/master.cf

Uncomment the following code blocks

```
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

and

```
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

## Edit and add stuff to /etc/postfix/main.cf

Change the values of these settings already configured in the file:

```
myhostname = mail.example1.com
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example1.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example1.com/privkey.pem
smtp_tls_CAfile = /etc/letsencrypt/live/mail.example1.com/cert.pem
```

Now add some new lines to the same file

```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

always_add_missing_headers = yes

home_mailbox = Mail/Inbox/
mailbox_command = /usr/lib/dovecot/deliver

virtual_alias_maps = hash:/etc/postfix/virtual
virtual_alias_domains = example2.com

tls_server_sni_maps = hash:/etc/postfix/vmail_ssl

recipient_delimiter = +
propagate_unmatched_extensions =
```

### Prepare the postfix hashtables

Add virtual address mappings to **/etc/postfix/virtual**

```
localuser@example2.com localuser
postmaster@example2.com postmaster
```

Optionally if you don't want any virutal mappings and want all the domains to use the same users, just add the secondary domains to the mydestination list in **/etc/postix/main.cf**

Now add the certificates to the SNI-map in **/etc/postfix/vmail_ssl**

```
mail.example1.com /etc/letsencrypt/live/mail.example1.com/privkey.pem /etc/letsencrypt/live/mail.example1.com/fullchain.pem
mail.example2.com /etc/letsencrypt/live/mail.example2.com/privkey.pem /etc/letsencrypt/live/mail.example2.com/fullchain.pem
```

### Generate the tables and restart postfix and dovecot

Finally generate the tables and reload postfix and dovecot

```sh
postmap /etc/postfix/virtual
postmap -F hash:/etc/postfix/vmail_ssl
systemctl restart dovecot
systemctl restart postfix
```

Now you should have a working setup for basic sending and receiving of email

Unfortunately, mostly due to spam and phishing emails, some additional configuration is required

## Configuring Spamassassin

### Add stuff to the postfix config

Head back over to **/etc/postfix/master.cf** and edit the following block to look like this

```
smtp      inet  n       -       y       -       -       smtpd
    -o content_filter=spamassassin
```

Additionally add this to the end of the file:

```
spamassassin unix -       n       n       -       -       pipe
  flags=R user=debian-spamd argv=/usr/bin/spamc -e /usr/sbin/sendmail -oi -f ${sender} ${recipient}
```

Optionally spamassassin can be further configured through its configuration file located in **/etc/spamassassin/local.cf** but
I won't be going into detail in configuring it.

### Enable spamassassin, start it and reload postfix

After running these commands spamassassin should be up and running

```
systemctl enable spamassassin
systemctl start spamassassin
systemctl reload postfix
```

## Setting up OpenDKIM

OpenDKIM helps you get through most of the spam filters :))

Setting it up is luckily quite trivial once you know what you're doing

### Add stuff to the configuration file

Add these lines to the configuration file located in **/etc/opendkim.conf**:

```
Domain example1.com example2.com
RequireSafeKeys false
Mode sv
KeyTable file:/etc/dkimkeys/keytable
SigningTable refile:/etc/dkimkeys/signingtable
InternalHosts refile:/etc/dkimkeys/trustedhosts
Socket inet:12301@localhost
```

### Configure the tables

Add these lines to these files

**/etc/dkimkeys/signingtable**:

```
*@example1.com default._domainkey.example1.com
*@example2.com default._domainkey.example2.com
```

**/etc/dkimkeys/keytable**:

```
default._domainkey.example1.com example1.com:default:/etc/dkimkeys/example1.com/default.private
default._domainkey.example2.com example2.com:default:/etc/dkimkeys/example2.com/default.private
```

**/etc/dkimkeys/trustedhosts**:

```
127.0.0.1
::1
*.example1.com
*.example2.com
```

### Generate keys and add the DNS records

Do the following for all the domains:

```
mkdir /etc/dkimkeys/example1.com
opendkim-genkey -b 1024 -d example1.com -D /etc/dkimkeys/example1.com -s default -v
chown opendkim:opendkim -R /etc/dkimkeys
```

The records can be found under **/etc/dkimkeys/example1.com/default.txt** and should be added to the DNS as follows:

```
TXT default._domainkey <contents_of_default.txt>
```

### Connect postfix to opendkim

The final step is to connect opendkim with postfix

This is done by adding the following lines into **/etc/postfix/main.cf**:

```
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:12301
non_smtpd_milters = $smtpd_milters
```

### Restart opendkim and reload postfix

After this all should be up and running.

```sh
systemctl restart opendkim
systemctl reload postfix
```

## Instructions for use

You should be able to connect to the mailserver from any of the domain using the mail. subdomain and the port 933 for IMAP and 587 for smtp

I've personally tested this setup on (neo)Mutt as well as on thunderbird

A user account can be added by adding a user on the system and adding the user to the mail group.

The password will be the users password which they use for loggin in.

## Conclusion

Now your brand new mailserver should be configured.

If something doesn't seem to work or there's some kind of inaccuracy in this article, don't hesitate to contact me via email. We'll figure out the solution together :)
