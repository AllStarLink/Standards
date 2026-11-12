# ASL004 - Node Key-Based Authentication

**Number:** ASL004<br/>
**Title:** Node Key-Based Authentication<br/>
**Author(s):** Jason McCormick<br/>
**Status:** Draft<br/>

## Abstract

This document outlines the process to authenticate the calling node
of an AllStarLink IAX2 call between nodes using a key-based mechanism
rather than relying on IP address mappings.

## Status

**DRAFT** - This document is open for comment and incompatible
changes are acceptable.

## Terminology

*Carrier-Grade NAT (CGNAT)* - A large-scale Network Address Translation deployment operated by a service provider at the network edge (rather than by the end user) that maps many subscribers' private IPv4 addresses to a smaller pool of shared public IPv4 addresses, typically to extend IPv4 address availability during the transition to IPv6.

*IAX2 Call* -  A "node link" in `app_rpt` is an IAX2 call to Asterisk with special semantics handled as SMS text messages within the channel. This term is used throughout this document to refer to all IAX2 traffic between nodes that is not related to node registration. It is chosen purposefully to avoid mixing `app_rpt` concepts of "node linking" from the technical details of making IAX2-based connections between Asterisk servers. All "node links" are, by definition, an "IAX2 call" because `app_rpt` linking is layered on top of one.

*Node Registration* - An AllStarLink node's authentication to the AllStarLink infrastructure. Successful registration in legacy mode updates an IPv4 address for the nodelist. Successful registration in modern mode still accomplishes IPv4 address updates, but will also update an IPv6 entry and publish the node's public key for node-to-node communications.

*Calling Node* - The node that originates an IAX2 call.

*Called Node* - The node that receives an IAX2 call.

*Modern Node* - A node that implements this standard.

*Legacy Node* - A node that does not implement this standard and
validates incoming IAX2 calls by IPv4 address only.

*Keyed Call* - An IAX2 call in which the calling node authenticates with
its node key as described in this standard.

*Legacy Call* - An IAX2 call made as the IAX2 user `radio` with no
IAX2-level authentication, validated by the called node by IPv4 address.

*Node Key* - The Ed25519 key pair belonging to one node number.

## Background

Since the inception of AllStarLink, the process to validate or authenticate IAX2 calls between nodes has been the mapping of an IPv4 address to each node number. For outgoing calls, this mapping was used as the directory to translate the called node number to the IP endpoint hosting the node. For incoming calls, this mapping was used to identify and validate the incoming node number being properly from that IP. This mapping also includes a UDP port that is the target to which the IAX2 call packets would be sent. This mapping is used regardless of the incoming UDP port of the other endpoint's traffic. This method leads has the following problems on the modern Internet:

1. The current structure completely precludes the use of IPv6 addresses for calls.

2. The fixed UDP port maps, ignoring the incoming UDP source port, requires manual configuration of traffic passing through a NAT device (i.e. port forwarding) rather than relying on natural state/session tracking.

3. The use of IPv4 address mapping for respond traffic, ignoring perceived IP addressing, breaks completely in CGNAT situations because distributed IPv4 addresses from the registration servers often will differ from the usable addresses between node connections.

4. Failed validation of an IAX2 call - coming from an "incorrect" or "unknown" IPv4 address - creates an indeterminate problem in the IAX2 protocol in that a calling node cannot know if it did not receive traffic due to incorrect/broken authentication (a local problem) or the remote system is offline or not functioning properly (a remote problem). This results in an inability to generate a useful error message/status for failed calls.

## Objectives

This standard shall outline a new mechanism for IAX2 calls between nodes that accomplishes the following:

1. Node Registration stores an IPv4 address, an IPv6 address, and the node's public key for node-to-node communication. Node registration shall be bimodal in that a registration shall be made for both an IPv4 and IPv6 address. This will be accomplished by two different registrations, one using IPv4 and one using IPv6. This will avoid problems with IPv4 "final address" detection issues due to NAT and problems with ephemeral IPv6 addressing schemes if such are employed on a Linux host.

2. Outbound IAX2 calls from a node shall exclusively use DNS resolution of `A`, `AAAA`, `SRV`, and `TXT` records to determine destination addresses, ports, and key capability of calls (unless node destinations are explicitly defined in `rpt.conf`).

3. Outbound IAX2 calls shall use IPv6 addressing if mutually available between callers, otherwise IPv4 shall be used.

4. Add an Ed25519 challenge/response authentication method to IAX2. The RSA method native to IAX2 is not suitable (see IAX2 Protocol Addition of ED25519).

5. Create `res_asl_keymgr`, which generates and stores each local node's key pair, retrieves other nodes' public keys from DNS, manages them within `res_crypto`, and acts as an Asterisk Realtime Architecture (ARA) backend that supplies `chan_iax2` with an on-demand IAX2 user definition matched 1:1 to a calling node and its key. This replaces the earlier idea of a `res_config_sqlite3` backend; no local database is kept in sync.

6. Establish a "fallback" call mechanism in `app_rpt` to try a legacy call if the keyed call fails.

7. A node that only places calls shall not need a well-known, forwarded, or unfirewalled UDP port. A node that wishes to receive calls continues to need a reachable, configured UDP port; this standard does not attempt NAT traversal for called nodes.

The following are explicitly not objectives of this revision:

* Authentication of the called node to the calling node. The calling node
continues to trust DNS for the destination, as it does today.

* Encryption of IAX2 calls.

* NAT traversal (port discovery, hole punching, or relaying) for called nodes.

## Technical Specifications

### Use of Keys

* Keys are Ed25519. A public key is 32 bytes, encoded as 44 characters of standard base64. A signature is 64 bytes, encoded as 88 characters of standard base64.

* There is one key pair per node number. A server hosting several node numbers holds one key pair for each.

* The key pair is generated by the node. The private key never leaves the host and is never known to the AllStarLink portal or registration servers.

* `res_asl_keymgr` shall generate a key pair at startup for any node number configured for registration that does not have one. The private key is stored in the Asterisk key directory as `asl-node-NODE.key` with file mode `0600`, owned by the Asterisk user. The key name within `res_crypto` is `asl-node-NODE`.

* The public key is sent with every registration and published in DNS by the AllStarLink infrastructure (see Modern Node Registration). Whatever key the node currently holds is therefore what the network publishes; a
deleted, lost, or regenerated key corrects itself at the next registration with no action in the portal.

* A node key is rotated by removing the private key file and restarting or reloading `res_asl_keymgr`. Until the new public key is visible in DNS (one registration interval, plus up to 75 seconds of DNS replication, plus the 60 second record TTL), keyed calls from the node will fail verification at called nodes and the node will fall back to legacy calls. An implementation may avoid this gap by continuing to sign with the old key until it observes the new key in DNS.

* A called node trusts the public key returned by DNS without further validation. This is the same level of trust placed in DNS by the legacy IPv4 address match. The `nodes.allstarlink.org` zone is DNSSEC-signed and
an operator may configure a validating resolver, but this standard does not require it.

### Modern Node Registration

Modern registration uses the HTTPS registration mechanism (`res_rpt_http_registrations`). Registration over IAX2 cannot carry a key and is always a legacy registration.

A modern node makes two registrations per interval when it has both address families available, one over IPv4 and one over IPv6. Both carry the same content.

The per-node object of the registration request gains two optional elements:

* `pubkey` - the node's public key as the string `ed25519:` followed by the
base64 public key.

* `callable` - a boolean, default `true`. A node that only places calls
sets this to `false`.

```json
{
    "port": 4569,
    "data": {
        "nodes": {
            "63001": {
                "node": "63001",
                "passwd": "STRING",
                "remote": 0,
                "pubkey": "ed25519:BASE64",
                "callable": true
            }
        }
    }
}
```

The registration response gains a `pubkey` element echoing the key now
stored for the node, in the same format, so that the node can detect and
log a mismatch.

The registration server shall:

1. Store the `pubkey` against the node number upon a successfully authenticated registration. A changed key replaces the stored key.

2. Remove the stored key when a successfully authenticated registration for the node does not include `pubkey`. A node that has reverted to legacy software must not continue to advertise a key.

3. Reject a `pubkey` that is not `ed25519:` followed by exactly 44 characters of valid base64, without failing the registration itself.

The DNS service shall publish the key as a `TXT` record separate from the existing informational `TXT` record of the node:

```text
_aslkey.63001.nodes.allstarlink.org.  60  IN  TXT  "v=ASL1; k=ed25519; p=BASE64"
```

* `v` is the record version and shall be `ASL1`.
* `k` is the key type and shall be `ed25519`.
* `p` is the base64 public key.

A consumer shall ignore a record whose `v` or `k` it does not recognize and treat the node as having no key.

The existence of a valid `_aslkey` record is the advertisement that a node is a modern node. It is used for both directions: a calling node checks it to decide whether to place a keyed call to the called node, and a called node retrieves it to verify a calling node.

When a node registers with `callable` set to `false`, the DNS service shall publish the `_aslkey` record but shall not publish the `SRV`, `A`, or `AAAA` records for the node. A calling node that finds an `_aslkey` record and no `SRV` record shall report that the node does not accept calls rather than attempting a call and waiting for a timeout.

### IAX2 Outgoing Call Lookup

For a called node not explicitly defined in `rpt.conf`, the calling node shall resolve:

1. `SRV` for `_iax._udp.NODE.nodes.allstarlink.org` to obtain the target host name and UDP port.

2. `A` and/or `AAAA` for the `SRV` target, according to Dual-Stack Network Behavior below.

3. `TXT` for `_aslkey.NODE.nodes.allstarlink.org`.

If the called node has a valid `_aslkey` record, the calling node places a keyed call. The IAX2 username is the calling node's own node number and the outbound key is that node's key:

```text
IAX2/MYNODE:[asl-node-MYNODE]@ADDRESS:PORT/CALLEDNODE
```

If the called node has no valid `_aslkey` record, the calling node places a legacy call directly, with no keyed attempt:

```text
IAX2/radio@ADDRESS:PORT/CALLEDNODE
```

A node explicitly defined in `rpt.conf` is called exactly as defined.

### Dual-Stack Network Behavior

* A calling node queries `AAAA` only if it has a non-loopback, non-link-local IPv6 address, and `A` only if it has a non-loopback, non-link-local IPv4 address.

* If both the calling node and the called node have IPv6 available, the call uses IPv6. Otherwise it uses IPv4.

* Key authentication is independent of the address family and of the source address and port of the call. The same node key and the same `_aslkey` record serve IPv4 and IPv6.

* The `SRV` port applies to both address families of a called node and is the configured, well-known port of that node.

* The calling node may send from any source address and port. Responses follow the path of the call, so a node that only places calls needs no port forwarding and no inbound firewall rule.

### IAX2 Protocol Addition of ED25519

The RSA authentication native to IAX2 is not used because the Asterisk implementation is fixed at RSA-1024 with SHA-1, and because the signature is carried in a single information element limited to 255 bytes, which
cannot hold the 344 character base64 signature of an RSA-2048 key. The existing challenge is also a single random integer of about 31 bits, and the signature covers only that challenge.

The following additions are made to IAX2 in `chan_iax2` and `res_crypto`:

* A new authentication method bit, `IAX_AUTH_ED25519`, value `0x08`, in the `AUTHMETHODS` information element.

* A new information element, `IAX_IE_ED25519_RESULT`, proposed value `0x3A`, a string carrying the base64 signature. See Additions to the IAX2 Specification below for the choice of value.

* When `IAX_AUTH_ED25519` is among the methods offered in an `AUTHREQ`, the `CHALLENGE` information element shall be 32 random bytes encoded as 44 characters of base64.

* The calling side signs, and the called side verifies, the following octet string, where `0x00` is a single zero octet separator:

    ```text
    "IAX2-ED25519-AUTH-v1" 0x00 CHALLENGE 0x00 USERNAME 0x00 CALLEDNUMBER
    ```

    `CHALLENGE` is the challenge string exactly as carried in the
    `AUTHREQ`. `USERNAME` is the IAX2 username of the `NEW`, which is the
    calling node number. `CALLEDNUMBER` is the called number of the `NEW`,
    which is the called node number.

* `res_crypto` gains support for loading, generating, signing with, and verifying with Ed25519 keys, and for adding a public key at run time.

The calling and called node numbers are included in the signed data so that a signature produced for a call to one node cannot be relayed by that node to authenticate as the calling node somewhere else.

### Additions to the IAX2 Specification

This section lists what must be added to the IAX2 specification (RFC 5456) and its IANA registry, "Inter-Asterisk eXchange (IAX) Parameters", for the Ed25519 method. It is written to stand apart from AllStarLink: nothing in it depends on node numbers, DNS, or `app_rpt`.

#### Registry Assignments

| Registry | Value | Name | Description |
| --- | --- | --- | --- |
| IAX Authentication Methods | `0x0008` | ED25519 | Ed25519 challenge/response |
| IAX Information Elements | `0x3A` | ED25519 RESULT | Ed25519 challenge result |

The authentication method value is the next unused bit of the `AUTHMETHODS` bit mask after MD5 (`0x0002`) and RSA (`0x0004`). It is unassigned in the IANA registry and unused in Asterisk.

The information element value is chosen to be free in both the IANA registry and Asterisk. The two differ, and a value taken from the registry alone can collide with a deployed Asterisk element:

| Values | IANA registry (last updated 2011-12-07) | Asterisk |
| --- | --- | --- |
| `0x1D`-`0x1E`, `0x20`-`0x25` | Reserved for future use | Provisioning, device type, service identifier, and firmware elements |
| `0x39` | Unassigned | `CALLINGANI2` |
| `0x3A`-`0x3F` | Unassigned | Unused |
| `0x40`-`0x46` | Assigned (T.38) | Unused |
| `0x47`-`0xFF` | Unassigned | Unused |

`0x39` is the lowest unassigned value in the registry and must not be requested. The request to the registry should also ask that `0x39` be recorded as in use by Asterisk.

#### Authentication Method

* A peer that supports the method may set `0x0008` in the `AUTHMETHODS` information element of an `AUTHREQ`, alone or together with other methods.

* When `0x0008` is set, the `CHALLENGE` information element of that `AUTHREQ` shall be 32 octets from a cryptographically secure random source, encoded as base64 (RFC 4648 section 4, with padding), giving a 44 character string. The same challenge serves any other method offered in the same `AUTHREQ`.

* A peer that receives an `AUTHREQ` without `0x0008` set shall not send an `ED25519 RESULT`.

* The method is defined for the `AUTHREQ`/`AUTHREP` exchange of a call. Its use in the `REGAUTH`/`REGREQ` exchange of a registration is not defined by this revision.

#### ED25519 RESULT Information Element

* Carried in an `AUTHREP`.

* The data is the 64 octet Ed25519 signature (RFC 8032, pure Ed25519, not Ed25519ph or Ed25519ctx), encoded as base64 (RFC 4648 section 4, with padding), giving an 88 character string. It is not null-terminated on the wire.

* The signed message is the following octet string, where `0x00` is a single zero octet:

    ```text
    "IAX2-ED25519-AUTH-v1" 0x00 CHALLENGE 0x00 USERNAME 0x00 CALLEDNUMBER
    ```

  * `"IAX2-ED25519-AUTH-v1"` is that literal string in ASCII, without quotes or terminator.
  * `CHALLENGE` is the data of the `CHALLENGE` information element of the `AUTHREQ`, exactly as received.
  * `USERNAME` is the data of the `USERNAME` information element of the `NEW` that began the call.
  * `CALLEDNUMBER` is the data of the `CALLED NUMBER` information element of that `NEW`. If the `NEW` carried none, it is empty.

* The verifying peer builds the same message from its own record of the challenge it issued and the `NEW` it received, and verifies the signature with the public key it holds for `USERNAME`. How that public key is obtained is outside the IAX2 specification.

* A failed verification is handled as any other failed authentication: the call is rejected with a `REJECT`.

#### Differences From the RSA Method

These are the points where the Ed25519 method deliberately departs from the existing RSA method, and which the specification text needs to state:

1. The challenge has a required minimum strength. The RSA and MD5 methods place no requirement on the challenge.

2. The signature covers the username and called number in addition to the challenge. The RSA method signs the challenge alone.

3. The signed message begins with a fixed label, so that a signature made for IAX2 authentication cannot be mistaken for a signature made with the same key for any other purpose, and so that a later revision can change the message format by changing the label.

### Node-to-Node Authentication

Called node configuration: `extconfig.conf` maps the `iaxusers` realtime family to `res_asl_keymgr`.

When an IAX2 `NEW` arrives with a username that matches no configured IAX2 user, `chan_iax2` asks the realtime backend for it. For a username consisting only of digits, `res_asl_keymgr` shall:

1. Resolve `TXT` for `_aslkey.USERNAME.nodes.allstarlink.org`.

2. If a valid key is found, make it available in `res_crypto` as `asl-node-USERNAME` and return a user definition equivalent to:

    ```conf
    type = user
    auth = ed25519
    inkeys = asl-node-USERNAME
    context = radio-secure
    callerid = "" <USERNAME>
    requirecalltoken = yes
    setvar = ASL_AUTH=key
    ```

3. If no valid key is found, return nothing. `chan_iax2` then treats the username as unknown.

Usernames that are not all digits are not looked up.

`chan_iax2` makes this request as a lookup of `name` equal to the username with `host` equal to `dynamic`. The backend shall answer that form of the request; it need not answer the lookups by address that `chan_iax2` tries afterward.

`chan_iax2` then issues the Ed25519 challenge, and verifies the result against the key as described above. A call that fails verification is rejected with an explicit `REJECT` and cause, so that a calling node can distinguish an authentication failure from an unreachable node.

Identity binding:

* The `callerid` of the returned user definition forces the caller ID number of the call to the authenticated username. A node therefore cannot authenticate as one node number and present another. All existing uses of the caller ID number as the calling node number, in `app_rpt` and in the dialplan (`NODENUM`, allowlist, denylist), continue to work unchanged.

* The caller ID name is cleared by the same mechanism. `app_rpt` uses the caller ID name only for calls whose caller ID number begins with `0` (IAX2 client and phone connections), never for node-to-node calls.

* The channel variable `ASL_AUTH` is set from the called node's own user definition and cannot be supplied by the calling node. When `ASL_AUTH` is `key`, `app_rpt` shall not perform the IPv4 address match of the calling
node. When it is absent, `app_rpt` performs the legacy match.

`res_asl_keymgr` shall cache lookup results, both positive and negative, for no longer than the record TTL, and shall limit the rate of lookups. The realtime lookup runs in an IAX2 processing thread and can be reached
by a `NEW` that has not yet passed call token validation, so an uncached, unlimited lookup would allow spoofed packets to generate DNS queries and stall call processing.

### Legacy Authentication Compatibility

The method used for a call is determined by the calling node from DNS before the call is placed:

| Calling node | Called node | Call placed | Validation by called node |
| --- | --- | --- | --- |
| Modern | Modern | Keyed | Ed25519 signature |
| Modern | Legacy | Legacy | IPv4 address match |
| Legacy | Modern | Legacy | IPv4 address match |
| Legacy | Legacy | Legacy | IPv4 address match |

Fallback: if a keyed call is rejected or is not answered, the calling node shall retry once as a legacy call. This covers a called node that has just reverted to legacy software and a calling node whose new key has not yet reached DNS. The retry shall not count against the retry limit of the link.

A legacy node that receives a keyed call does not know the numeric username and responds with an `AUTHREQ` that does not offer `IAX_AUTH_ED25519`. The calling node cannot answer it and ends the attempt immediately, so the fallback does not wait for a timeout.

Limitations during the transition:

* The benefits for nodes behind CGNAT, and for nodes whose registered IPv4 address otherwise differs from the address their calls arrive from, apply only when the called node is a modern node. A legacy call from such a node to a legacy node fails the IPv4 address match exactly as it does today.

* IPv6 is used only between two modern nodes.

* While legacy calls are accepted, a party able to send from a node's registered IPv4 address can still call as that node. Key authentication does not make this worse, and closing it is a step in the transition timeline below.

## Transition Timeline

Dates are to be determined. The intended sequence is:

1. Modern nodes register keys, place keyed calls to modern nodes, and accept both keyed and legacy calls.

2. A modern called node refuses a legacy call that claims a node number for which an `_aslkey` record exists. From this point a modern node cannot be impersonated to another modern node by address.

3. Legacy calls are no longer accepted, and the IPv4 address match is removed. (MAYBE? Years later?)

## Open Issues

1. Registration of the values in Additions to the IAX2 Specification with IANA, and acceptance of the Ed25519 method by the Asterisk project. If it is not accepted upstream it must be carried as a patch in the AllStarLink Asterisk packages. The Asterisk comparison was made against 22.11 only. A separate, unrelated request to IANA for an Ed25519 method is understood to exist using information element `0x20`, which Asterisk already uses as `DEVICETYPE`.

2. The `app_rpt` change that redials as a legacy call without consuming the link's retry budget is not yet designed.

3. Whether the dual-stack destination syntax of the pending IPv6 change accepts the `USER:[KEY]@` form has not been confirmed.

4. Whether `res_asl_keymgr` can hand a public key to `res_crypto` in memory, or must write a `.pub` file to the key directory.

5. The registration server and DNS service changes (`pubkey` storage, `callable`, the `_aslkey` record, suppression of `SRV`/`A`/`AAAA`) have not been reviewed against the current implementations.

6. Whether step 2 of the transition timeline needs its own notice period, since it changes the behavior of nodes that are already modern.
