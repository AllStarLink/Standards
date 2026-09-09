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


* Creation of OTU tokens will be limited to a maximum of 10 **live** (i.e.
    issued and not yet consumed) tokens per `client-id` plus `username`
    combination. Issuance of an 11th live token for a given pair will cause
    the oldest live token for that pair to be expired immediately, and so
    forth. A token is removed from this live count as soon as it is
    consumed by a validation request, not merely when its TTL elapses -
    see Rate Limiting Implementation below.

* Limits on token creation:

    * Creation of OTU tokens will be limited per `client-id` to 1 per second.
    Requests beyond 1 per second for up to 10 seconds is considered a soft
    limit and will be met with an HTTP 429 response. Persistent soft limit
    exhaustion for 20 seconds will result in a 5 minute `client-id`-level
    block.

    * Regardless of `client-id`, creation of OTU tokens will be limited
    per IP address to 10 per second. Requests beyond 1 per second for up to 10 seconds is considered a soft
    limit and will be met with an HTTP 429 response. Persistent soft limit
    exhaustion for 20 seconds will result in a 5 minute `client-id`-level
    block.

* Limits on token validation:

    * Authentication validation requests will be limited to a maximum of 10
    per second per IP on a sliding window basis.  Requests
    beyond 10 per second for up to 10 seconds is considered a soft limit and
    will be met with an HTTP 429 response. Persistent soft limit exhaustion
    for 20 seconds will result in a 5 minute IP-level block.

* General limits:

    * In addition to the endpoint-specific limits above, all AA requests
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

#### Rate Limiting Implementation
To satisfy the limits above with a minimum of Redis round-trips, the token-creation
and token-validation endpoints shall perform all rate-limit checks and token/list
bookkeeping as a single atomic operation per request via a server-side Lua script
(invoked through `redis.Redis.register_script()` / `EVALSHA`, not as separate
`GET`/`INCR`/`SET` calls from the application).

**Keys used:**

| Key | Type | Purpose | TTL |
|---|---|---|---|
| `token:<sha256_hex>` | string | the OTU token itself (`username:client-id`) | 86400s |
| `rl:create:cid:<client-id>` | string (counter) | per-`client-id` creation rate | 1s window |
| `rl:create:ip:<ip>` | string (counter) | per-IP creation rate | 1s window |
| `rl:abuse:ip:<ip>` | string (counter) | combined creation+validation abuse counter | 1s window |
| `rl:validate:ip:<ip>` | string (counter) | per-IP validation rate | 1s window |
| `block:cid:<client-id>` | string (flag) | active 5-minute block | 300s |
| `block:ip:<ip>` | string (flag) | active 5-minute block | 300s |
| `tokens:<client-id>:<username>` | list | hashes of the up-to-10 live tokens for this pair | none (self-managed) |

**Token-creation script**, atomically:

1. Checks `block:cid:<client-id>` and `block:ip:<ip>`; if either exists, returns `BLOCKED`.
2. `INCR`/`EXPIRE`s `rl:create:cid:<client-id>`, `rl:create:ip:<ip>`, and `rl:abuse:ip:<ip>`;
   if any exceeds its threshold, returns `SOFT_LIMIT` (and, if the persistent-exhaustion
   window is also exceeded, sets the relevant `block:*` key with a 300s TTL and returns
   `BLOCKED` instead).
3. `LPUSH`es the new token hash onto `tokens:<client-id>:<username>`; if the list length
   exceeds 10, `RPOP`s the evicted hash and `DEL`s its `token:<evicted_hash>` entry, per
   the 10-live-token cap.
4. `SET`s `token:<sha256_hex>` with the new value and 86400s TTL.
5. Returns `OK`.

**Token-validation script**, atomically:

1. Increments/expires `rl:validate:ip:<ip>` and `rl:abuse:ip:<ip>`; if either exceeds its
   threshold, returns `SOFT_LIMIT` / `BLOCKED` following the same logic as creation.
2. `GETDEL`s `token:<sha256_hex>` to retrieve and remove the stored value, then splits it
   on the first `:` to recover `username` and `client-id`.
3. `LREM`s the consumed hash from `tokens:<client-id>:<username>` (the list is capped at
   10 entries, so this is an O(10) operation regardless of load) so the live-token count
   reflects only currently unconsumed tokens rather than the last 10 issued.
4. Returns the recovered `username` (or `BLOCKED`/`SOFT_LIMIT` from step 1).

The application layer (Python, via `redis-py`) registers both scripts once at process
startup and calls them with the relevant identifiers as arguments, translating `OK` →
HTTP 200/`status:true`, `SOFT_LIMIT` → HTTP 429, and `BLOCKED` → HTTP 429 (blocked).

**Known limitation:** a token that is never consumed and never evicted by new issuance
remains in the `tokens:<client-id>:<username>` list until its own `token:<hash>` key
naturally expires via TTL (86400s) - nothing proactively prunes the list on passive key
expiry. This means the live-token count can briefly overcount (treating an
expired-but-unpruned entry as live) for up to one day in the low-traffic case where
fewer than 10 tokens are issued in that window. This is considered acceptable and
self-correcting; exactly solving it would require a keyspace-notification listener
process, which is not justified by this edge case.

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
matches or not, the OTU token is consumed, which also removes it from
the live-token count for its `client-id`/`username` pair (see Rate
Limiting Implementation above).

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

## Appendix: Reference Lua Scripts
These implement the atomic behavior described in Rate Limiting Implementation
above. Both scripts take the current unix timestamp as an `ARGV`, supplied by
the Python caller (e.g. `int(time.time())`), rather than calling Redis `TIME`
internally, so behavior is independent of replication mode. Each script is
registered once at process startup via `redis.Redis.register_script()` and
invoked thereafter by its `SHA1` (`EVALSHA`). The two scripts duplicate a
small rate-check helper rather than sharing it; a production deployment on
Redis/Valkey 7+ could instead load that helper once as a Redis Function
(`FUNCTION LOAD`) and call it from both scripts.

### `token_create.lua`
```lua
-- Atomically applies all token-creation rate limits, enforces the
-- 10-live-token cap per client-id+username, and stores the new token.
--
-- KEYS[1] = block:cid:<client-id>
-- KEYS[2] = block:ip:<ip>
-- KEYS[3] = rl:create:cid:<client-id>
-- KEYS[4] = rl:create:ip:<ip>
-- KEYS[5] = rl:abuse:ip:<ip>
-- KEYS[6] = tokens:<client-id>:<username>
-- KEYS[7] = token:<sha256_hex>        -- the new token's key
--
-- ARGV[1]  = now                      -- unix timestamp (seconds)
-- ARGV[2]  = token_hash                -- the sha256 hex digest (list entry)
-- ARGV[3]  = token_value               -- "<username>:<client-id>"
-- ARGV[4]  = client_id_limit           -- 1   (per second)
-- ARGV[5]  = ip_create_limit           -- 10  (per second)
-- ARGV[6]  = ip_abuse_limit            -- 20  (per second)
-- ARGV[7]  = persist_seconds           -- 20  (continuous violation before block)
-- ARGV[8]  = block_ttl                 -- 300 (5 minutes)
-- ARGV[9]  = token_cap                 -- 10  (live tokens per client-id+username)
-- ARGV[10] = token_ttl                 -- 86400
--
-- Returns: "OK" | "SOFT_LIMIT" | "BLOCKED"

local now             = tonumber(ARGV[1])
local token_hash      = ARGV[2]
local token_value     = ARGV[3]
local client_id_limit = tonumber(ARGV[4])
local ip_create_limit = tonumber(ARGV[5])
local ip_abuse_limit  = tonumber(ARGV[6])
local persist_seconds = tonumber(ARGV[7])
local block_ttl       = tonumber(ARGV[8])
local token_cap       = tonumber(ARGV[9])
local token_ttl       = tonumber(ARGV[10])

local block_cid_key = KEYS[1]
local block_ip_key  = KEYS[2]

-- Already blocked? Bail out without touching any counters.
if redis.call('EXISTS', block_cid_key) == 1 then
    return 'BLOCKED'
end
if redis.call('EXISTS', block_ip_key) == 1 then
    return 'BLOCKED'
end

local RANK = { OK = 0, SOFT_LIMIT = 1, BLOCKED = 2 }

-- Generic 1-second rate check with persistent-violation escalation.
-- Returns "OK", "SOFT_LIMIT", or "BLOCKED" (and sets block_key on escalation).
local function check_rate(rate_key, limit, block_key)
    local count = redis.call('INCR', rate_key)
    if count == 1 then
        redis.call('EXPIRE', rate_key, 1)
    end

    if count <= limit then
        redis.call('DEL', rate_key .. ':viol')
        return 'OK'
    end

    local viol_key = rate_key .. ':viol'
    local viol_start = redis.call('GET', viol_key)
    if not viol_start then
        redis.call('SET', viol_key, now, 'EX', persist_seconds)
        return 'SOFT_LIMIT'
    end

    if (now - tonumber(viol_start)) >= persist_seconds then
        redis.call('SET', block_key, '1', 'EX', block_ttl)
        redis.call('DEL', viol_key)
        return 'BLOCKED'
    end

    return 'SOFT_LIMIT'
end

local worst = 'OK'
local r1 = check_rate(KEYS[3], client_id_limit, block_cid_key)
local r2 = check_rate(KEYS[4], ip_create_limit, block_ip_key)
local r3 = check_rate(KEYS[5], ip_abuse_limit,  block_ip_key)
if RANK[r1] > RANK[worst] then worst = r1 end
if RANK[r2] > RANK[worst] then worst = r2 end
if RANK[r3] > RANK[worst] then worst = r3 end

if worst ~= 'OK' then
    return worst
end

-- Enforce the 10-live-token cap for this client-id + username pair.
local tokens_key = KEYS[6]
redis.call('LPUSH', tokens_key, token_hash)
if redis.call('LLEN', tokens_key) > token_cap then
    local evicted = redis.call('RPOP', tokens_key)
    if evicted then
        redis.call('DEL', 'token:' .. evicted)
    end
end

-- Store the new token itself.
redis.call('SET', KEYS[7], token_value, 'EX', token_ttl)

return 'OK'
```

### `token_validate.lua`
```lua
-- Atomically applies validation-path rate limits, retrieves and consumes
-- the OTU token, and prunes it from the live-token cap list.
--
-- KEYS[1] = block:ip:<ip>
-- KEYS[2] = rl:validate:ip:<ip>
-- KEYS[3] = rl:abuse:ip:<ip>
-- KEYS[4] = token:<sha256_hex>        -- the token being validated
--
-- ARGV[1] = now                       -- unix timestamp (seconds)
-- ARGV[2] = ip_validate_limit         -- 10 (per second)
-- ARGV[3] = ip_abuse_limit            -- 20 (per second)
-- ARGV[4] = persist_seconds           -- 20
-- ARGV[5] = block_ttl                 -- 300
--
-- Returns:
--   "SOFT_LIMIT" | "BLOCKED"          -- rate-limited; token untouched
--   "NOTFOUND"                        -- token didn't exist / already consumed
--   "<username>:<client-id>"          -- success; caller splits on first ":"
--
-- NOTE: tokens:<client-id>:<username> is only knowable after the token
-- value is retrieved, so it is built and accessed dynamically below rather
-- than passed via KEYS. This is safe on a single Redis/Valkey instance; a
-- clustered deployment would need a hash-tag scheme (e.g.
-- tokens:{<client-id>}:<username>) so this key and KEYS[4] always resolve
-- to the same slot.

local now               = tonumber(ARGV[1])
local ip_validate_limit = tonumber(ARGV[2])
local ip_abuse_limit    = tonumber(ARGV[3])
local persist_seconds   = tonumber(ARGV[4])
local block_ttl         = tonumber(ARGV[5])

local block_ip_key = KEYS[1]

if redis.call('EXISTS', block_ip_key) == 1 then
    return 'BLOCKED'
end

local RANK = { OK = 0, SOFT_LIMIT = 1, BLOCKED = 2 }

local function check_rate(rate_key, limit, block_key)
    local count = redis.call('INCR', rate_key)
    if count == 1 then
        redis.call('EXPIRE', rate_key, 1)
    end

    if count <= limit then
        redis.call('DEL', rate_key .. ':viol')
        return 'OK'
    end

    local viol_key = rate_key .. ':viol'
    local viol_start = redis.call('GET', viol_key)
    if not viol_start then
        redis.call('SET', viol_key, now, 'EX', persist_seconds)
        return 'SOFT_LIMIT'
    end

    if (now - tonumber(viol_start)) >= persist_seconds then
        redis.call('SET', block_key, '1', 'EX', block_ttl)
        redis.call('DEL', viol_key)
        return 'BLOCKED'
    end

    return 'SOFT_LIMIT'
end

local worst = 'OK'
local r1 = check_rate(KEYS[2], ip_validate_limit, block_ip_key)
local r2 = check_rate(KEYS[3], ip_abuse_limit, block_ip_key)
if RANK[r1] > RANK[worst] then worst = r1 end
if RANK[r2] > RANK[worst] then worst = r2 end

if worst ~= 'OK' then
    return worst
end

local stored = redis.call('GETDEL', KEYS[4])
if not stored then
    return 'NOTFOUND'
end

local sep = string.find(stored, ':')
local username  = string.sub(stored, 1, sep - 1)
local client_id = string.sub(stored, sep + 1)

if client_id ~= nil and client_id ~= '' then
    local tokens_key = 'tokens:' .. client_id .. ':' .. username
    -- KEYS[4] is "token:<hash>"; strip the "token:" prefix to recover the hash.
    local token_hash = string.sub(KEYS[4], 7)
    redis.call('LREM', tokens_key, 0, token_hash)
end

return stored
```