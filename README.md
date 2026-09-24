# AllStarLink Standards

This repository contains the documented standards for the AllStarLink
Network that are used in the following ways:

1. Registration of app_rpt to the AllStarLink Network
2. Directory services provided by the AllStarLink Network to clients
3. Statistics APIs for app_rpt to publish statistics
4. Other APIs
5. Other administrative policies that have community relevance

This repository will also serve as the official status tracker of
supported features, APIs, and other structures.

## Standards

| Name | Status | Description |
| --- | --- | --- |
| ASL001-Node_Key-Based_Authentication | *Draft* | Description of using public-private keypairs for node authentication |
| ASL002-App_Authentication | *Fatal Flaw Review* | Technical specification of the Application Authentication (AA) method |
| ASL003-AI_Use_Practice | *Active* | Practices on use of AI-contributed code and information to AllStarLink |

## Taxonomy

Taxonomy of a standards document is simple. Each document will
have a filename starting with `ASLnnn-` followed by the title
of the document with spaces converted to `_` and then appended
with `.md`.

## Status

| Status | Description |
| --- | --- |
| *Draft* | Under active development by the author; may change at any time and should not be implemented |
| *Fatal Flaws Review* | Fixed-length review period in which only blocking issues (security, interoperability, feasibility, conflicts) are considered |
| *Accepted* | Passed review and frozen; implementers may build against it, but it is not yet deployed or in effect |
| *Active* | In force on the AllStarLink Network; changes require a new revision or a superseding standard |
| *Deprecated* | Still in effect but being phased out; new implementations should not adopt it |
| *Superseded* | Replaced by a newer standard, which is referenced in the document |
| *Withdrawn* | Abandoned by the author before acceptance |
| *Rejected* | Did not pass review due to an unresolved fatal flaw |

## Fatal Flaws Review

A Fatal Flaws Review is the final check before a standard is
accepted. It is not a general editing pass; the goal is to find
problems serious enough that the standard must not be adopted
as written.

When the author considers a Draft complete, the standard moves to
*Fatal Flaws Review* and the review is announced with a fixed end
date. During the review period, anyone may raise an issue. Each
issue is classified as either a fatal flaw or a non-blocking comment.

A fatal flaw is one of the following:

1. A security or privacy weakness
2. A break in interoperability or backward compatibility with
   deployed nodes, clients, or services
3. Text that cannot be implemented as written because it is
   ambiguous, self-contradictory, or depends on something that
   does not exist
4. A conflict with another *Accepted* or *Active* standard
5. Content that is outside the scope of AllStarLink or conflicts
   with established project policy

Everything else, such as wording, formatting, style, or alternative
approaches of similar merit, is a non-blocking comment. The author
may incorporate non-blocking comments at their discretion without
affecting the review.

Outcomes of the review:

* If no fatal flaws remain open, the standard moves to *Accepted* or, in cases where there is no implementation time needed, directly to *Active*.
* If a fatal flaw is resolved by an editorial change, the review
  continues without interruption.
* If resolving a fatal flaw requires a substantive change to the
  requirements of the standard, the review period restarts from
  the beginning.
* If a fatal flaw cannot be resolved, the standard is either
  returned to *Draft* for rework or marked *Rejected*.

