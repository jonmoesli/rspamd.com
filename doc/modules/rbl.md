---
layout: doc
title: RBL module
---
# RBL module
{:.no_toc}

The RBL module provides support for checking various messages elements, such as senders IP addresses, URLs, Emails, Received headers chains, SMTP data (such as HELO domain) and so on, against the set of Runtime Black Lists (RBL) usually provided by means of dedicated DNS zones.

By default, Rspamd comes with a set of RBL rules configured to the popular resources that are usually free for non-profit usage (including fair usage policies). Should you require a different level of support and access then please contact the relevant vendors.

For example, you can use [Abusix Mail Intelligence](https://docs.abusix.com/105726-setup-abusix-mail-intelligence/rspamd-configuration) or [Spamhaus DQS](https://github.com/spamhaus/rspamd-dqs) or any other RBL provider that suits your needs. 

<div id="toc" markdown="1">
  <h2 class="toc-header">Contents</h2>
  * TOC
  {:toc}
</div>

## Configuration structure

Configuration for this module is structured as following:

~~~ucl
# local.d/rbl.conf

# 'rbls' subsection under which the RBL definitions are nested
rbls {
  # rbl-specific subsection
  an_rbl {
    ## required settings
    # checks to enable for this RBL
    checks = ["from"];
    # Address used for RBL-testing
    rbl = "rbl.example.net";

    ## some optional settings
    # Explicitly defined symbol
    symbol = "SOME_SYMBOL";

    # redefined defaults for IPv6 only RBL
    ipv4 = false;
    ipv6 = true; # Define IPv6 only RBL

    # Possible responses from RBL and symbols to yield
    returncodes = {
      # Name_of_symbol = "address";
      EXAMPLE_ONE = "127.0.0.1";
      EXAMPLE_TWO = "127.0.0.2";
    }
  }
}
~~~

## Configuration parameters

### Global parameters

- `local_exclude_ip_map`: map containing additional IPv4/IPv6 addresses/subnets that should be considered private and excluded from checks where `exclude_local` is `true` (the default).

### RBL-specific parameters

The required parameters `rbl` and `checks` set the address used for testing and the checks to be performed respectively. Valid values for `checks` can include any of the following:

- `content_urls` - URLs extracted by `lua_content` (eg. from PDFs)
- `dkim` - domain that provided DKIM signature for a message
- `emails` - email addresses found in a message-body
- `from` - the sending IP that sent the message
- `helo` - HELO provided by the sender
- `rdns` - sender's hostname as provided to Rspamd (expected to be forward-confirmed)
- `received` - IP addresses found in `Received` headers
- `replyto` - address from the `Reply-To` header of a message
- `urls` - URLs extracted from message body

Selectors can be used to lookup up arbitrary data; see the section on selectors for more information.

~~~ucl
# /etc/rspamd/local.d/rbl.conf
rules {
  # minimal configuration example
  SIMPLE_RBL {
    rbl = "rbl.example.net";
    checks = ["from"];
  }
}
~~~

Optional parameters (and their defaults if applicable) are as follows:

- `dkim_domainonly` (true) - lookup eSLD associated with DKIM signature rather than full label
- `dkim_match_from` (false) - only check DKIM signatures matching the `From` header
- `emails_domainonly` (false) - lookup domain of address instead of full address
- `enabled` (true) - allow for disabling of RBLs
- `exclude_local` (true) - do not check private addresses in this RBL
- `exclude_users` (false) - do not check this RBL if sender is an authenticated user
- `hash` - valid for `helo` and `emails` RBL types - lookup hashes instead of literal strings. Possible values for this parameter are `sha1`, `sha256`, `sha384`, `sha512` and `md5` or any other value for the default hashing algorithm.
- `hash_format` - encoding to use for hash: `hex`, `base32` or `base64`
- `ignore_whitelist` (false) - allow whitelists to neutralise this RBL
- `images` (false) - whether image URLs should be checked by `urls` check
- `ipv4` (true) - if IPv4 addresses should be checked
- `ipv6` (true) - if IPv6 addresses should be checked
- `is_whitelist` (false) - denotes that this RBL is an whitelist
- `local_exclude_ip_map` - map containing IPv4/IPv6 addresses/subnets that shouldn't be checked in RBLs (where `exclude_local` is `true` (default)).
- `monitored_address` (`1.0.0.127`) - fixed address to check for absence; see section on monitoring for more information
- `no_ip` (false) - do not look up IP addresses in this RBL
- `requests_limit` (9999) - maximum number of entities extracted by URL checks
- `resolve_ip` - resolve the domain to IP address
- `returnbits` - dictionary of symbols mapped to bit positions; if the bit in the specified position is set the symbol will be returned
- `returncodes` - dictionary of symbols mapped to lua patterns; if result returned by the RBL matches the pattern the symbol will be returned
- `selector_flatten` (false) - lookup result of chained selector as a single label
- `selector` - one or more selectors producing data to look up in this RBL; see section on selectors for more information
- `unknown` (false) - yield default symbol if `returncodes` or `returnbits` is specified and RBL returns unrecognised result
- `whitelist_exception` - for whitelists; list of symbols which will not act as whitelists

Some examples of using RBL:

~~~ucl
rbls {

    blocklist {
      symbol = "BLOCKLIST";
      rbl = "blocklist.bl";
      checks = ['from', 'received'];
    }
    
    WHITELIST_BASE {
      checks = ['from', 'received'];
      is_whitelist = true;
      rbl = "whitelist.wl";
      symbol = "WL_RBL_UNKNOWN";
      unknown = true;
      returncodes = {
        "WL_RBL_CODE_2" = "127.0.0.2";
        "WL_RBL_CODE_3" = "127.0.0.3";
      }
    }
      
    DNS_WL {
      symbol = "DNS_WL";
      rbl = "dnswl.test";
      checks = ['dkim'];
      dkim_domainonly = false;
      dkim_match_from = true;
      ignore_whitelist = true;
      unknown = false;

      returncodes {
        DNS_WL_NONE = "127.0.%d+.0";
        DNS_WL_LOW = "127.0.%d+.1";
        DNS_WL_MED = "127.0.%d+.2";
        DNS_WL_HI = "127.0.%d+.3";
        DNS_WL_BLOCKED = "127.0.0.255";
      }
    }
  }
  
~~~

## URL rules

From version 2.0, both `Emails` and `SURBL` modules are deprecated in honor of the rules for RBL module. Old rules are
automatically converted by Rspamd on start. If you have your custom rules in either `SURBL` or `Emails` module then they are
converted in a way to have priority over RBL modules to allow smooth migration. However, the new rules should be written for
RBL module only as `SURBL` and `Emails` modules transition phase will not last forever.

`SURBL` module was previously responsible for scanning of URLs found in messages against a list of known RBLs. However, these functions are now transferred to this module.

## URL rules configuration

By default, Rspamd defines a set of URL lists in the configuration. However, their terms
of usage normally disallows commercial or very extensive usage without purchasing
a specific sort of license.

Nonetheless, they can be used by personal services or low volume requests free
of charge.

Here are the default lists specified:

~~~ucl
# local.d/rbl.conf
# 'surbl' subsection under which the SURBL definitions are nested
surbl {
# List of domains that are not checked by surbl
whitelist = "file://$CONFDIR/local.d/maps.d/surbl-whitelist.inc.local";

  rules {
    "SURBL_MULTI" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "multi.surbl.org";
      checks = ['emails', 'dkim', 'urls'];
      emails_domainonly = true;
      urls = true;

      returnbits = {
        CRACKED_SURBL = 128; # From February 2016
        ABUSE_SURBL = 64;
        MW_SURBL_MULTI = 16;
        PH_SURBL_MULTI = 8;
        SURBL_BLOCKED = 1;
      }
    }
    
    "URIBL_MULTI" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "multi.uribl.com";
      checks = ['emails', 'dkim', 'urls'];
      emails_domainonly = true;

      returnbits = {
        URIBL_BLOCKED = 1;
        URIBL_BLACK = 2;
        URIBL_GREY = 4;
        URIBL_RED = 8;
      }
    }
    
    "RSPAMD_URIBL" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "uribl.rspamd.com";
      checks = ['emails', 'dkim', 'urls'];
      # Also check images
      images = true;
      # Сheck emails for URLs
      emails_domainonly = true;
      # Hashed BL
      hash = 'blake2';
      hash_len = 32;
      hash_format = 'base32';

      returncodes = {
        RSPAMD_URIBL = [
          "127.0.0.2",
        ];
      }
    }
    
    "DBL" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "dbl.spamhaus.org";
      no_ip = true;
      checks = ['emails', 'dkim', 'urls'];
      emails_domainonly = true;

      returncodes = {
        # spam domain
        DBL_SPAM = "127.0.1.2";
        # phish domain
        DBL_PHISH = "127.0.1.4";
        # malware domain
        DBL_MALWARE = "127.0.1.5";
        # botnet C&C domain
        DBL_BOTNET = "127.0.1.6";
        # abused legit spam
        DBL_ABUSE = "127.0.1.102";
        # abused spammed redirector domain
        DBL_ABUSE_REDIR = "127.0.1.103";
        # abused legit phish
        DBL_ABUSE_PHISH = "127.0.1.104";
        # abused legit malware
        DBL_ABUSE_MALWARE = "127.0.1.105";
        # abused legit botnet C&C
        DBL_ABUSE_BOTNET = "127.0.1.106";
        # error - IP queries prohibited!
        DBL_PROHIBIT = "127.0.1.255";
        # issue #3074
        DBL_BLOCKED_OPENRESOLVER = "127.255.255.254";
        DBL_BLOCKED = "127.255.255.255";
      }
    }
    
    "SEM_URIBL_UNKNOWN" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "uribl.spameatingmonkey.net";
      no_ip = true;
      checks = ['emails', 'dkim', 'urls'];
      emails_domainonly = true;
      returnbits {
        SEM_URIBL = 2;
      }
    }
  }
~~~

Each list should have a `suffix` parameter that defines the list itself and optionally for some replies processing logic
either by `returnbits` or `returncodes` sections.

Since some URL lists do not accept `IP` addresses, it is also possible to disable sending of URLs with IP address in the host to such lists. That could be done by specifying `no_ip = true` option:

~~~ucl
"DBL" {
    rbl = "dbl.spamhaus.org";
    # Do not check numeric URLs
    no_ip = true;
}
~~~

It is also possible to check DKIM signatures domains, HTML images URLs and email addresses (domain part) in mail's body part for URLs by using URL blacklists:

~~~ucl
    "RSPAMD_URIBL" {
      ignore_defaults = true; # for compatibility with old defaults
      rbl = "uribl.rspamd.com";
      checks = ['emails', 'dkim', 'urls'];
      # Also check images
      images = true;
      # Сheck emails for URLs
      emails_domainonly = true;
      # Hashed BL
      hash = 'blake2';
      hash_len = 32;
      hash_format = 'base32';

      returncodes = {
        RSPAMD_URIBL = [
          "127.0.0.2",
        ];
      }
    }
~~~

In this example, we also enable privacy for requests by hashing all elements before sending. It is supported by a limited number of RBLs (e.g. Rspamd URL blacklist or by MSBL EBL).

## Monitoring

By default, Rspamd checks each RBL rule to be a valid DNS list as defined in [RFC 5782](https://datatracker.ietf.org/doc/html/rfc5782). This is done to avoid situations when some RBL blacklists the whole world or is irresponsive. For the IP based rules (meaning that an IP address is queried), Rspamd will query for `127.0.0.1` address as, per RFC, this must return `NXDOMAIN` response. However, some DNS lists are non RFC compatible, so you can disable monitoring for them as following:

~~~ucl
    "HOSTKARMA_URIBL" {
      rbl = "hostkarma.junkemailfilter.com";
      no_ip = true;
      enabled = false;
      
      returncodes = {
        URIBL_HOSTKARMA_WHITE = "127.0.0.1";
        URIBL_HOSTKARMA_BLACK = "127.0.0.2";
        URIBL_HOSTKARMA_YELLOW = "127.0.0.3";
        URIBL_HOSTKARMA_BROWN = "127.0.0.4";
        URIBL_HOSTKARMA_NOBLACK = "127.0.0.5";
        URIBL_HOSTKARMA_24_48H = "127.0.2.1";
        URIBL_HOSTKARMA_LAST_10D = "127.0.2.2";
        URIBL_HOSTKARMA_OLDER_10D = "127.0.2.3";
      }
      disable_monitoring = true;
    }
~~~

For non IP lists (DKIM, URL, Email and so on), Rspamd will just produce some long random string to query expecting that this random string will *very likely* return `NXDOMAIN` by its nature.


## Principles of operation

In this section, we define how `RBL` module performs its checks.

### TLD composition

By default, we want to check some top level domain, however, many domains contain
two components while others can have 3 or even more components to check against the
list. By default, rspamd takes top level domain as defined in the [public suffixes](https://publicsuffix.org).
Then one more component is prepended, for example:

    sub.example.com -> [.com] -> example.com
    sub.co.uk -> [.co.uk] -> sub.co.uk

However, sometimes even more levels of domain components are required. In this case,
the `exceptions` map can be used. For example, if we want to check all subdomains of
`example.com` and `example.co.uk`, then we can define the following list:

    example.com
    example.co.uk

Here are new composition rules:

    sub.example.com -> [.example.com] -> sub.example.com
    sub1.sub2.example.co.uk -> [.example.co.uk] -> sub2.example.co.uk
    
### Specific URL composition rules

From Rspamd 2.5, there is also support for a custom composition rules per RBL rules. This is provided by `lua_urls_compose` library. Here is a basic explanation of the composition rules used:

```lua
-- First one is the input hostname, the second is the expected results
cases = {
  {'example.com', 'example.com'},
  {'baz.example.com', 'baz.example.com'},
  {'3.baz.example.com', 'baz.example.com'},
  {'bar.example.com', 'example.com'},
  {'foo.example.com', 'foo.example.com'},
  {'3.foo.example.com', '3.foo.example.com'},
}
-- Just a domain means domain + 1 level
-- *.domain means the full hostname if the last part matches
-- !domain means exclusion
-- !*.domain means the same in fact :)
-- More rules can be added easily...
local excl_rules1 = {
  'example.com',
  '*.foo.example.com',
  '!bar.example.com'
}
```

To define a specific map for these rules one can use the following syntax

~~~ucl
# local.d/rbl.conf
rules {
  EXAMPLE_RBL = {
      suffix = "example.url.bl.com";
      url_compose_map = "${CONFDIR}/maps.d/url_compose_map.list";
      checks = ['emails', 'dkim', 'urls'];
      emails_domainonly = true;
      ignore_defaults = true;
  }
}
~~~

Where in maps you can use something like this:

```
*.dirty.sanchez.com
!not.dirty.sanchez.com
41.black.sanchez.com
```

So it will check 5 hostname components for all urls in `dirty.sanchez.com` (e.g. `sub.some.dirty.sanchez.com` will be transformed to just `some.dirty.sanchez.com`) but not for `not.dirty.sanchez.com` where the normal tld rules will apply (e.g. `some.not.dirty.sanchez.com` -> `sanchez.com`), and for `41.black.sanchez.com` all 5 components will be checked, e.g. `something.41.black.sanchez.com`.

### DNS composition

SURBL module composes the DNS request of two parts:

- TLD component as defined in the previous section;
- DNS list suffix

For example, to form a request to multi.surbl.org, the following applied:

    example.com -> example.com.multi.surbl.com

### Results parsing

Normally, DNS blacklists encode reply in A record from some private network
(namely, `127.0.0.0/8`). Encoding varies from one service to another. Some lists
use bits encoding, where a single DNS list or error message is encoded as a bit
in the least significant octet of the IP address. For example, if bit 1 encodes `LISTA`
and bit 2 encodes `LISTB`, then we need to perform bitwise `OR` for each specific bit
to decode reply:

     127.0.0.3 -> LISTA | LISTB -> both bit symbols are added
     127.0.0.2 -> LISTB only
     127.0.0.1 -> LISTA only

This encoding can save DNS requests to query multiple lists one at a time.

Some other lists use direct encoding of lists by some specific addresses. In this
case you should define results decoding principle in `ips` section not `bits` since
bitwise rules are not applicable to these lists. In `ips` section you explicitly
match the IP returned by a list and its meaning.

## IP lists

From rspamd 1.1 it is also possible to do two step checks:

1. Resolve IP addresses of each URL
2. Check each IP resolved against SURBL list

In general this procedure could be represented as following:

* Check `A` or `AAAA` records for `example.com`
* For each IP address resolve it using reverse octets composition: so if IP address of `example.com` is `1.2.3.4`, then checks would be for `4.3.2.1.uribl.tld`

## Disabling rules

Rules can be disabled by setting the `enabled` setting to `false`. This allows for easily disabling SURBLs without overriding the full default configuration. The example below could be added to `/etc/rspamd/local.d/surbl.conf` to disable the `RAMBLER_URIBL` URIBL.

~~~ucl
rules {
  "RAMBLER_URIBL" {
    enabled = false;
  }
}
~~~

## Use of URL redirectors

SURBL module is designed to work with [url_redirector module](./url_redirector.html) which can help to resolve some known redirectors and extract the real URL to check with this module. Please refer to the module's documentation about how to work with it. SURBL module will automatically use that results.

## Selectors

Selectors can be used to look up arbitrary values in RBLs.

The `selector` field could be configured as a string in the rule settings if only one selector is needed:

~~~
checks = ["replyto"];
selector = "from('mime'):addr";
hash = "sha1";
symbols_prefixes {
  replyto = "REPLYTO";
  selector = "FROM";
}
~~~

Or they could be specified as a map of user-specified names to selectors if more than one is needed:

~~~
selector = {
  mime_domain = "from('mime'):domain";
  subject_digest = "header('subject').lower.digest('hex')";
}
symbols_prefixes {
  mime_domain = "MIME_DOMAIN";
  subject_digest = "SUBJECT_DIGEST"
}
~~~
