# ASL001 - Node Key-Based Authentication

**Number:** ASL001<br/>
**Title:** Node Key-Based Authentication<br/>
**Author(s):** Jason McCormick<br/>
**Status:** Draft<br/>

## Abstract
This document outlines the process to perform mutual authentication
of AllStarLink IAX2 calls between nodes using a key-based mechanism
rather than relying on IP address mappings.

## Status
**DRAFT** - This document is open for comment and incompatible
changes are acceptable.

## Terminology

*Carrier-Grade NAT (CGNAT)* - A large-scale Network Address Translation deployment operated by a service provider at the network edge (rather than by the end user) that maps many subscribers' private IPv4 addresses to a smaller pool of shared public IPv4 addresses, typically to extend IPv4 address availability during the transition to IPv6.

*IAX2 Call* -  A "node link" in `app_rpt` is an IAX2 call to Asterisk with special semantics handled as SMS text messages within the channel. This term is used throughout this document to refer to all IAX2 traffic between nodes that is not related to node registration. It is chosen purposefully to avoid mixing `app_rpt` concepts of "node linking" from the technical details of making IAX2-based connections between Asterisk servers. All "node links" are, by definition, an "IAX2 call" because `app_rpt` linking is layered on top of one.

*Node Registration* - An AllStarLink node's authentication to the AllStarLink
infrastructure. Successful registration in legacy mode updates an IPv4 address
for the nodelist. Successful registration in modern mode still accomplishes
IPv4 address updates, but will also update an IPv6 entry and retrieve key
material for node-to-node communications.

## Background
Since the inception of AllStarLink, the process to validate or authenticate IAX2 calls between nodes has been the mapping of an IPv4 address to each node number. For outgoing calls, this mapping was used as the directory to translate the called node number to the IP endpoint hosting the node. For incoming calls, this mapping was used to identify and validate the incoming node number being properly from that IP. This mapping also includes a UDP port that is the target to which the IAX2 call packets would be sent. This mapping is used regardless of the incoming UDP port of the other endpoint's traffic. This method leads has the following problems on the modern Internet:

1. The current structure completely precludes the use of IPv6 addresses for calls.

2. The fixed UDP port maps, ignoring the incoming UDP source port, requires manual configuration of traffic passing through a NAT device (i.e. port forwarding) rather than relying on natural state/session tracking.

3. The use of IPv4 address mapping for respond traffic, ignoring perceived IP addressing, breaks completely in CGNAT situations because distributed IPv4 addresses from the registration servers often will differ from the usable addresses between node connections.

4. Failed validation of an IAX2 call - coming from an "incorrect" or "unknown" IPv4 address - creates an indeterminate problem in the IAX2 protocol in that a calling node cannot know if it did not receive traffic due to incorrect/broken authentication (a local problem) or the remote system is offline or not functioning properly (a remote problem). This results in an inability to generate a useful error message/status for failed calls.

## Objectives
This standard shall outline a new mechanism for IAX2 calls between nodes that accomplishes the following:

1. Node Registration stores an IPv4 address, an IPv6 address, and retrieves private key material for node-to-node communication. Node registration shall be bimodal in that a registration shall be made for both an IPv4 and IPv6 address. This will be accomplished by two different registrations, one using IPv4 and one using IPv6. This will avoid problems with IPv4 "final address" detection issues due to NAT and problems with ephemeral IPv6 addressing schemes if such are employed on a Linux host.

2. Outbound IAX2 calls from a node shall exclusively use DNS resolution of `A`, `AAAA`, and `SRV` records to determine destination addresses and ports of calls (unless node destinations are explicitly defined in `rpt.conf`).

3. Outbound IAX2 calls shall use IPv6 addressing if mutually available between callers, otherwise IPv4 shall be used.

4. Create a `res_config` use framework that permits the creation of on-demand IAX peername stanzas to match 1:1 from a node to its key material as an IAX named context. Backend ARA (Asterisk Realtime Architecture) would be res_config_sqlite3. These dynamic contexts would use RSA-based challenge/response authentication native to IAX2 and also support calltokens properly. Note: `res_config` is a dynamic configuration module for Asterisk that is part of the core system. This would insert and remove IAX2 peer contexts on demand from the `chan_iax` stack. See https://docs.asterisk.org/Fundamentals/Asterisk-Configuration/Database-Support-Configuration/Realtime-Database-Configuration/.

5. Create `res_asl_keymgr` that handles public keys from directories and manages items within `res_crypto`.

6. Establish a "fallback" call mechanism in `app_rpt` to try the existing/legacy IAX call context type if the RSA-based context call fails.

## Technical Specifications

### Use of Keys


### Modern Node Registration
TODO

### IAX2 Outgoing Call Lookup
TODO

### Dual-Stack Network Behavior
TODO

### IAX2 Protocol Addition of ED25519

### Node-to-Node Authentication
TODO

### Legacy Authentication Compatibility
TODO

## Transition Timeline




