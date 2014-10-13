# Sign DKIM signature on outgoing emails for new mail domain

[TOC]

> Don't know where Amavisd config file is? check this tutorial:
> [Locations of configuration and log files of mojor components](file.locations.html#amavisd).

iRedMail configures Amavisd to sign outgoing emails for the first mail domain
you added during iRedMail installation. If you added new mail domain, you
should update Amavisd config file to sign DKIM signature for it.

Let's say your first mail domain added during iRedMail installation is
`mydomain.com`, and new mail domain is `new_domain.com`, please follow below
steps to enable DKIM signing for outgoing emails of this domain.

## Use existing DKIM key for new mail domain

if you already have a working DKIM and valid DKIM DNS record, it's ok to
use this existing DKIM key. This way, you don't need to ask your customer
who owns this new domain to add DKIM DNS record.

* Find below setting in Amavisd config file `amavisd.conf`:

```
dkim_key('mydomain.com', "dkim", "/var/lib/dkim/mydomain.com.pem");

@dkim_signature_options_bysender_maps = ( {
    ...
    "mydomain.com"  => { d => "mydomain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    ...
});
```

Add one line in `@dkim_signature_options_bysender_maps`, after `"mydomain.com"`
line like below:

```
@dkim_signature_options_bysender_maps = ( {
    ...
    "mydomain.com"  => { d => "mydomain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    "new_domain.com"  => { d => "mydomain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    ...
});
```

* Restart Amavisd service.

## Generate new DKIM key for new mail domain

If you or your customer prefer to use their own DKIM key, you can generate
a new DKIM key and ask your customer to add DKIM DNS record. Refer to our
tutorial to [add DKIM DNS record](setup_dns.html#dkim-record-for-your-mail-domain-name).

* Generate new DKIM key for new domain.

```shell
# amavisd-new genrsa /var/lib/dkim/new_domain.com.pem
```

* Find below setting in Amavisd config file `amavisd.conf`:

```
dkim_key('mydomain.com', "dkim", "/var/lib/dkim/mydomain.com.pem");
```

Add one line after above line like below:

```
dkim_key('new_domain.com', "dkim", "/var/lib/dkim/new_domain.com.pem");
```

* Find below setting in Amavisd config file `amavisd.conf`:

```
@dkim_signature_options_bysender_maps = ( {
    ...
    "mydomain.com"  => { d => "mydomain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    ...
});
```

Add one line after `"mydomain.com"` line like below:

```
@dkim_signature_options_bysender_maps = ( {
    ...
    "mydomain.com"  => { d => "mydomain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    "new_domain.com"  => { d => "new_domain.com", a => 'rsa-sha256', ttl => 10*24*3600 },
    ...
});
```

* Restart Amavisd service.

Again, don't forget to ask your customer to add DKIM DNS record.

## Use one DKIM key for all mail domains without updating Amavisd config file

For compatibility with dkim_milter the signing domain can include a '*'
as a wildcard - this is not recommended as this way amavisd could produce
signatures which have no corresponding public key published in DNS.
The proper way is to have one dkim_key entry for each mail domain.

If you still want to try this, please follow below steps:

* Find below setting in Amavisd config file `amavisd.conf`:

```
dkim_key('mydomain.com', "dkim", "/var/lib/dkim/mydomain.com.pem");
```

* Replace it by below line:

```
dkim_key('*', "dkim", "/var/lib/dkim/mydomain.com.pem");
```

* Restart Amavisd serivce.

With above setting, all outbound emails with be signed with this dkim key.
And Amavisd will show a warning message when start amavisd service:

> dkim: wildcard in signing domain (key#1, *), may produce unverifiable
> signatures with no published public key, avoid!

## See also

* Don't know what DKIM is? Check our tutorial here:
  [What is a DKIM DNS record](setup_dns.html#dkim-record-for-your-mail-domain-name).