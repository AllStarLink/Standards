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
mobile and desktop applications to the AllStarLink network. These terms
are used today: "WebTransceiver mode", "WT auth", "WT Mode", etc.

This process has several fundamental flaws that make it unsuitable for
modern Internet-based applications.

1. There is no session security of any form. The "credential" is passed
over unencrypted IAX with no replay protections

2. The "credential" is not collision-resistant

## Objectives
This standard shall outline a new mechanism for IAX2 calls between an application
and a normal node that accomplishes the following:

1. Retain a node-less connection type for software applications connecting to full AllStarLink nodes

2. Co-exist for a period of time with the existing authentication choices

3. Retire remaining "WebTransceiver" infrastructure from www.allstarlink.org

4. Improve connection security

## Technical Specifications
Application Authentication (AA) will work on a one-time-use (OTU) token mechanism. Each
token shall be provided by an application during an AA session. Once
a node has used/verified the token, the token will be deleted as part of the
use/retrieval. Any unconsumed token will expire after 86400 seconds (1 day).

Basic specifications:

* All strings unless otherwise specified shall be a maximum of 64 characters
in length

* All numbers/integers are 64-bit unsigned integers unless otherwise specified

* Applications implementing AA shall generate a UUIDv4 once, at install
or initial configuration time, and persist it for the lifetime of that
installation. This value is required as the `client-id` field on token
requests (see below). It is client-asserted and used solely as a
rate-limiting and telemetry signal - it is not an authentication
credential and MUST NOT be treated as a trusted or verified identity.

* Creation of OTU tokens will be limited per `client-id` to 1 per second.
Requests beyond 1 per second for up to 10 seconds is considered a soft
limit and will be met with an HTTP 429 response. Persistent soft limit
exhaustion for 20 seconds will result in a 5 minute `client-id`-level
block.

* Regardless of `client-id`, creation of OUT tokens will be limited
per IP address to 10 per second. Requests beyond 1 per second for up to 10 seconds is considered a soft
limit and will be met with an HTTP 429 response. Persistent soft limit
exhaustion for 20 seconds will result in a 5 minute `client-id`-level
block.

* Creation of OTU tokens will be limited to a maximum of 10 tokens per
`client-id` plus `username` on a rolling basis- i.e., only 10 tokens will be stored per
`client-id` plus `usernaem` combination. Issuance of token #11 for
a given `client-id` will cause #1 to be expired, and so forth.

* Authentication validation requests will be limited to a maximum of 10
per second per IP on a sliding window basis.  Requests
beyond 10 per second for up to 10 seconds is considered a soft limit and
will be met with an HTTP 429 response. Persistent soft limit exhaustion
for 20 seconds will result in a 5 minute IP-level block.

* In addition to the endpoint-specific IP limits above, all AA requests
(token creation and validation, combined) from a single IP address are
subject to an overall abuse limit of 20 requests per second on a sliding
window basis, regardless of the `username` or `client-id` presented.
This is a backstop limit intended to catch distributed abuse (e.g.,
credential stuffing across many usernames, or `client-id` rotation)
that could otherwise stay under the per-username and per-`client-id`
thresholds above. Requests beyond this limit for up to 10 seconds is
considered a soft limit and will be met with an HTTP 429 response.
Persistent soft limit exhaustion for 20 seconds will result in a 5
minute IP-level block.

* All API calls will return HTTP 200 upon successful HTTP-level
and message-syntax correctness. Note: this means that an implementing
client must parse the response to determine if authentication has
succeeded or failed. Unlike register.allstarlink.org, this API
will not incorrectly return an HTTP 403 for a non-forbidden API call
regardless of the internal authentication status.

### Initial Authentication
An application shall POST to an endpoint at `https://api.allstarlink.org/TODO/appauth/request`. This endpoint
is callable unprivileged. The POST shall be a JSON document with three required
elements `username`, `password`, and `client-id`. The `username` and `password`
correspond to the username and password for the AllStarLink account.
The `client-id` is the UUIDv4 generated once by the application at
install/configuration time as described above; a request missing
`client-id` or containing a malformed value shall be rejected with a
`status` of `false`. `client-id` is stored alongside the token and used
only to apply the per-`client-id` rate limits described above; it has
no other effect on authentication success or failure and is never
returned in the response.

The API shall also accept a client-optional
`request-id`. The `request-id` field is ignored by the server, except that it
shall be included in the response when provided in the request.
The JSON request structure shall be:

```json
{
    "username": "STRING",
    "password": "STRING",
    "request-id": "STRING",
    "client-id": "STRING"
}
```

The response for an authentication request shall return a `status` as
a boolean and an `auth-token` as a string. A response where `status` is `true`
is a successful authentication; a return of `false` is unsuccessful. The `auth-token`
field shall return a SHA2-256 hex digest of a one-time-use token code upon
success or shall be `null` if the authentication is not successful.
The JSON response structure shall be:

```json
{
    "status": true | false,
    "auth-token": "STRING" | null,
    "request-id": "STRING"
}
```

### API Implementation Details

#### Storage
Upon successful authentication by a client using the AA method, a one-time-use token
shall be generated and stored in a Redis/Valkey database as a string value, so
that it can be atomically retrieved and removed with `GETDEL` (a hash type
cannot be used here, as `GETDEL` only operates on string keys). The key shall
store the associated username and `client-id`, joined by a `:` separator,
and shall be stored as:

```
SET token:<sha256_hex> "<username>:<client-id>" EX 86400
```

#### Retrieval / Validation
When the API endpoint for authentication is called at `https://api.allstarlink.org/TODO/appauth/validate`
the following shall happen:

1. [RV1] If the request is in the AA format of `?t=TOKEN&u=CALLSIGN` then
only the Redis/Valkey database shall be consulted for the TOKEN,
the CALLSIGN matched, and then a successful auth returned. When
retrieved from Redis/Valkey, the API shall use the `GETDEL` method
to retrieve the stored value and then delete the entry. The value
shall be split on the first `:` to recover the username (before) and
the `client-id` (after). The CALLSIGN match is performed against the
username only; `client-id` is not used in validation logic and is
retained only for audit/telemetry purposes. Regardless if the callsign
matches or not, the OTU token is consumed.

If the validation is successful, the return will be `OHYES` followed
by the callsign.

If the validation is unsuccessful, the return will be an empty string.

2. [RV2] If the request is in the AA format of `?t=TOKEN&u=` (i.e.
the `u=` parameter is empty) then it shall be assumed that it is
a modern node with a legacy WT client. In this case,
an attempt shall be made to retrieve the token with `GETDEL`, but
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
`CALLERID(name)` data element, the same way the WebTransceiver token is passed
today. The format of this shall be `TOKEN/CALLSIGN`. Upon sending an IAX2 call
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

The API `/api/v2/auth-wt-legacy.php` will be retired 180 days
following the implementation date.

### Validation Methods
Method **RV1** is the permanent, long-term API for Application Authentication
and will be supported from *Day D* of implementation.

Methods **RV2**, **RV3**, and **RV4** are considered deprecated as of the
day of implementation of this standard. Methods **RV2** and **RV4** shall be
supported by the API for *Day + 180* following implementation. Method **RV3** shall
be supported by the API for *Day + 366*. Additionally, at the end of the
one-year transitional period the API internal rewrite for `authwebphone.pl` will
be removed.