# ASL002 - App Authentication

**Number:** ASL002<br/>
**Title:** App Authentication<br/>
**Author(s):** Jason McCormick<br/>
**Status:** Draft<br/>

## Abstract
This document outlines the modern method for Application Authentication (AA).
This replaces the legacy "Web Transceiver" (WT) authentication.

## Status
**DRAFT** - This document is open for comment and incompatible changes
are acceptable.

## Background
Years ago, the allstarlink.org website hosted a Java Applet communicator
to interoperate with standard nodes on the AllStarLink Network. In the
intervening years, the authentication method developed for this applet
named "Web Transceiver" became the de facto standard for authenticating
mobile and desktop applications to the AllStarLink network. This has become,
today, terms like "WebTransceiver mode", "WT auth", etc.

This process has several fundamental flaws that make it unsuitable for
modern Internet-based applications.

1. There is no session security of any form. The "credential" is passed
over unencrypted IAX with no replay protections.

2. The "credential" is not collision-resistant.

## Objectives
This standard shall outline a new mechanism for IAX2 calls between and application
and a normal node that accomplishes the following:

1. Retain a node-less connection type for software applications connecting to full AllStarLink nodes

2. Co-exist for a period of time with the existing authentication choices

3. Retire remaining "WebTransceiver" infrastructure from www.allstarlink.org

4. Improve connection security

## Technical Specifications
Application Authentication (AA) will work on a one-time-use (OTU) token mechanism. Each
token shall be provided by an application during an AA auth session. Once
a node has used/verified the token, the token will be deleted as part of the
use/retrieval. Any unconsumed token will expire after 86400 seconds (1 day).

Basic specifications:

* All strings unless otherwise specified shall be a maximum of 64 characters
in length.

* All numbers/integers are 64-bit unsigned integers unless otherwise specified

* Creation of OTU tokens will be limited per IP address to 1 per second

* Creation of OTU tokens will be limited per Username to 1 per second

* Creation of OTU tokens will be limited to a maximum of 10 tokens per Username

### Initial Authentication
An application shall POST to an endpoint at `https://api.allstarlink.org/TODO/appauth/request`. This endpoint
is callable unprivileged. The POST shall be a JSON document with two required
elements `username` and `password`. These correspond to the username and password
for the AllStarlink account. The API shall also accept a client-optional
`request-id`. The `request-id` field is ignored by the server other than it
shall be included in the response when provided in the request.
The JSON request structure shall be:

```json
{
    username: "STRING",
    password: "STRING",
    response-id: "STRING"           # optional
}
```

The response for an an authentication request shall return a `status` as
an integer and an `auth-token` as a string. A response where `status` is `true`
is a successful authentication; a return of `false` is unsuccessful. The `auth-token`
field shall return a SHA2-256 hex digest of a one-time-use token code upon
success or shall be a `null` of the authentication as not successful.
The JSON response structure shall be:

```json
{
    status: true | false,
    auth-token: "STRING" | NULL
    response-id: "STRING"           # will not be returned if not present in the request
}
```

### API Implementation Details

#### Storage
Upon successful authentication by a client using the AA method, a one-time-use token
shall be generated and stored in a Redis/Valkey database. The tuple stored for
each token shall be expressed/stored as:

```
HSET token:<sha256_hex> username "<username>" created_at "<unixtime>"
EXPIRE token:<sha256_hex> 86400
```

#### Retrieval / Validation
When the API endpoint for authentication is called at `https://api.allstarlink.org/TODO/appauth/validate`
the following shall happen:

1. [RV1] If the request is in the AA format of `?t=TOKEN&u=CALLSIGN` then
only the Redis/Valkey database shall be consulted for the TOKEN,
the CALLSIGN matched, and then a successful auth returned. When
retrieved from Redis/Valkey, the API shall use the `GETDEL` method
to retrieve the token hashset and then delete the entry. Regardless
if the callsign matches or not, the OTU token is consumed.

If the validation is successful, the return will be `OHYES` followed
by the callsign.

If the validation is unsuccessful, the return will be an empty string.

2. [RV2] If the request is in the AA format of `?t=TOKEN&u=` (i.e.
the `u=` parameter is empty) then it shall be assumed that it is
a modern node with a legacy WT client. In this case,
the token shall be attempted to retrieved with `GETDEL` but
the callsign-matching will be skipped and the callsign associated
with the token will simply be returned following `OHYES`.

3. If the request is in the legacy format of `?TOKEN` then
the following shall be evaluated in order:

   a. [RV3] If the `TOKEN` is longer than 12 characters, then
   it shall be assumed that the request is from a legacy node
   with a modern client and the format is provided in
   `TOKEN/CALLSIGN`. The string will be parsed out, checked,
   and returned according to #1 above.

   b. [RV4] If the `TOKEN` is 12 characters, then it shall be assumed
   to be a legacy node with a legacy client. The current WebTransceiver
   process will be evaluated (substr(12) on the password hash).

4. Anything that is invalid or doesn't parse correctly according to
RV1 to RV4 will be returned as a "?".

### Application-Side Usage
In general, the application using the AA method must request a new OTU token
before making a connection to a node. The application may store the
username and password of the AllStarLink portal account at its discretion
and use that stored information to create as many OTU tokens as needed
subject to the overall rate-limits described above.

When connecting to a node over IAX, the callsign and token shall be passed in the
`CALLERID(name)` data element as the WebTransceiver token is currently. The
format of this shall be `TOKEN/CALLSIGN`. Upon sending an IAX2 call
to a node, the application should consider the token consumed and delete it,
regardless of whether or not the connection succeeds. Future calls to
the new node must retrieve and use a new key.

### Node-Side Usage
Upon implementation of this spec, `extensions.conf` shall be changed
for the `[allstar-public]` context such that the authentication test
now functions as follows:

```conf
same => n,Set(TOKEN=${CUT(CALLERID(name),/,1)})
same => n,Set(CALLSIGN=${CUT(CALLERID(name),/,2)})
same => n,Set(RESP=${CURL(https://api.allstarlink.org/TODO/appauth/validate?t=${TOKEN}&u=${CALLSIGN})})
same => n,GotoIf($["${RESP:0:1}" = "?"]?hangit)
same => n,GotoIf($["${RESP:0:1}" = ""]?hangit)
same => n,GotoIf($["${RESP:0:5}" != "OHYES"]?hangit)
same => n,Set(RCALLSIGN=${RESP:5})
same => n,GotoIf($["${RCALLSIGN}" != "${CALLSIGN}"]?hangit)
same => n,Set(NODENUM=${CALLERID(num)})
```

This change will also be emailed out to all node owners and posted
on the forum to make the corresponding change on their nodes. Additionally,
this change shall be made in all packages going forward and a one-shot
attempt made in the `asl3-asterisk-config.postinst` to correct the item
in `/etc/asterisk/extensions.conf`.

Legacy clients will be maintained during the transitional period
by internally rewriting `https://register.allstarlink.org/cgi-bin/authwebphone.pl?${TOKEN}`
to the new API system.

## Transition Timeline
Implementation of this standard shall start a transition timeline of
366 days.

### Authentication Methods
Legacy "Web Transceiver" authentication - hooking `login.php` and
calling `webtransceiver.php` will be supported for 180 days following
the implementation date. At that time, `webtransceiver.php` will be
removed and consumers of `login.php` shall be on notice that the
login process will be re-engineered in the near future.

The API `/api/v2/auth-wt-legacy.php` will be returned 180 days
following the implementation date.

### Validation Methods
Method **RV1** is the permanent, long-term API for Application Authentication
and will be support from *Day D* of implementation.

Methods **RV2**, **RV3**, and **RV4** are considered deprecated as of the
day of implementation of this standard. Methods **RV2** and **RV3** shall be
supported by the API for *Day + 180* following implementation. Method **RV3** shall
be supported by the API for *Day + 366*. Additionally, at the end of the
one year transitional period the API internal rewrite for `authwebphone.pl` will
be removed.