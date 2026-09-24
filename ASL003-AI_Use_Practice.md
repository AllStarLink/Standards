# ASL003 - AI Use Practice

**Number:** ASL003<br/>
**Title:** AI Use Practice<br/>
**Author(s):** Jason McCormick<br/>
**Status:** Active
**Effective:** 2026-09-24<br/>

## Abstract
This document outlines the acceptable practices for use of AI tools
in contributions to AllStarLink. AI-assisted contributions are welcome
provided the human contributor understands, can defend, and takes full
responsibility for what they submit, and that the contribution is free
of known license violations.

## Status
**Active** - n force on the AllStarLink Network; changes require a new revision or a superseding standard.

## Background
AI coding assistants and generative tools (collectively, "AI tools") are now
part of many contributors' workflows. They can improve productivity and lower
the barrier to entry, but they also introduce risks: plausible-but-wrong code,
unreviewed "drive-by" submissions, unclear provenance of generated output, and
increased review burden on maintainers.

Other leading open source projects have published guidance on the subject,
including the Linux Kernel (guidelines for tool-generated content and the
`Assisted-by:` tag), Asterisk, Gentoo, NetBSD, and QEMU. The approaches vary
from full bans to open acceptance, but converge on a common theme: the
contributor, not the tool, is accountable. This standard adopts that principle
while remaining welcoming to contributors who use AI responsibly.

## Objectives
1. Welcome AI-assisted contributions that meet the same quality bar as any other
contribution

2. Keep accountability with the human contributor

3. Protect the project from known license and copyright violations

4. Preserve maintainer time by discouraging unreviewed, low-effort, or
bulk machine-generated submissions

5. Provide transparency about AI involvement where it is material

## Technical Specifications

### Scope
This standard applies to all contributions to AllStarLink projects and
services, including but not limited to source code, tests, build and packaging
files, documentation, standards documents, issue reports, pull request
descriptions, code review comments, and forum or support content. It applies
to any AI tool, including code completion, chat assistants, and autonomous
agents.

### Principles
AI tools are treated like any other tool such as a compiler, editor, or
code generator. Their use is neither required nor prohibited. The following
principles apply to every AI-assisted contribution.

1. **The contributor is the author.** The person submitting a contribution is
solely responsible for it, regardless of how it was produced. AI tools cannot
be held accountable and are never listed as the author or copyright holder.

2. **Understanding is required.** A contributor MUST understand the change
they submit. This means being able to explain what it does, why it is needed,
how it works, and what alternatives were considered.

3. **Be able to defend it.** A contributor MUST be able to justify the
inclusion of every part of the contribution in review on its technical merits.
"The AI suggested it" is not a justification. If a reviewer asks why a piece of
code, design choice, or claim exists, the contributor must be able to answer
without deferring to the tool.

4. **Same standards for all contributions.** AI-assisted contributions are
held to the same standards for correctness, style, testing, security, and
maintainability as any other contribution. They receive no additional
scrutiny and no additional leniency solely because of AI involvement.

### Contributor Requirements
A contributor submitting AI-assisted work MUST:

1. Review the complete output line by line before submitting it, and remove
anything they do not understand or cannot justify.

2. Build, run, and test the change. Contributors SHOULD NOT submit code that has
not been compiled and exercised, and SHOULD include tests (functional or
documented/manual) for new behavior.

3. Verify all factual claims, API usage, function names, package names,
configuration options, and references. AI tools are known to fabricate these,
including dependencies that do not exist. Dependencies MUST be verified as real
and legitimately sourced before being added.

4. Comply with the project's existing coding style, contribution process,
and developer requirements.

5. Not include secrets, credentials, private keys, personal data, non-public
node or user information, or embargoed security details in prompts sent to
third-party AI services, nor in the resulting contribution.

6. Respond to review feedback themselves. Review discussion is between
humans. Contributors MUST NOT paste reviewer comments into an AI tool and
relay the unreviewed output as a reply, and MUST be able to make requested
changes with understanding of them.

### Licensing and Copyright
AI-generated output raises unresolved legal questions. To protect the project
and its users, contributors MUST:

1. Ensure that the contribution is compatible with the license of the project
it is submitted to, and that they have the right to submit it under that
license.

2. Not knowingly submit output that reproduces third-party code or content
that is incompatible with the project's license or that the contributor lacks
the right to contribute. If the tool reports a source, license, or attribution
for suggested output, or the output is recognizably copied from an existing
work, the contributor MUST honor that license and attribution or discard the
output.

3. Not submit output from tools whose terms of use restrict, or impose
conditions on, the use or licensing of generated output in a way that is
incompatible with the license of the project.

4. Not submit material that the contributor knows or has reason to believe
violates a license, copyright, patent, or other right.

A contributor is not required to prove a negative. The standard is knowledge
and good faith diligence: a contribution is acceptable when the contributor
has taken reasonable care and has no knowledge of a violation. When a
violation is later discovered, the contribution will be removed or rewritten
promptly.

### Disclosure and Attribution
Contributors SHOULD disclose AI involvement when it is material to the
contribution, such as when a significant portion of the code or text was
generated, or when an AI tool was used to find a vulnerability or design an
approach. Trivial uses such as autocomplete, spell checking, grammar
correction, translation, and boilerplate expansion do not require disclosure.

Where disclosure is made, contributors SHOULD use an `Assisted-by:` commit
trailer, following the convention adopted by the Linux Kernel:

```
Assisted-by: <tool name> <model or version>
```

For example:

```
Assisted-by: Claude claude-sonnet-5
```

Tool trailers are informational only. They do not transfer responsibility
away from the contributor.

Maintainers MAY ask a contributor whether and how AI tools were used when
it is relevant to reviewing the contribution. Contributors MUST answer
honestly. Concealing AI use when asked is a violation of this standard.

### Communications and Reports
Issues, bug reports, security reports, and review comments are held to the
same accountability standard as code.

1. Reports MUST be verified by the reporter. A report describing a problem the
reporter has not experienced or reproduced is not acceptable, even if it
is well-formatted and confident in tone.

2. Security vulnerability reports produced with AI assistance MUST include a
reproducible demonstration or clear evidence, and MUST be submitted through
the project's security reporting channel.

3. Text should be concise and written for the human reader. Contributors SHOULD
edit AI-generated descriptions and comments to remove filler and unverified
claims.

4. Autonomous or unattended agents MUST NOT open issues, post comments,
submit pull requests, or send messages to project channels without a human
reviewing the content first, and the human is responsible for it.

### Prohibited Practices
The following are not acceptable and MAY result in the contribution being
closed without detailed review:

1. Submitting AI output the contributor has not read, understood, or tested

2. Bulk, unsolicited, or "drive-by" changes generated by AI tools across
the codebase, such as mass refactors, style rewrites, "whitespace cleanup"
or automated "fix" submissions, without prior agreement from maintainers

3. Responding to review questions with tool output the contributor cannot
explain

4. Knowingly submitting output that violates a license or third-party rights

5. Misrepresenting the origin of a contribution or concealing AI use when
asked

6. Using AI tools to circumvent project processes, such as generating
fabricated benchmarks, test results, or reviews

### Maintainer Practices
Maintainers and reviewers:

1. MAY use AI tools to assist in review, triage, or drafting, subject to the
same responsibility, confidentiality, and accuracy requirements as
contributors. A maintainer remains accountable for any review decision.

2. MUST NOT send non-public contributions, security reports, or private
user information to third-party AI services without the consent of the
submitter.

3. MAY close a contribution that appears to be unreviewed AI output, or that
the contributor cannot explain or defend, with a brief explanation
referencing this standard.

4. SHOULD respond with courtesy. A first violation is normally handled by
explaining this standard and inviting the contributor to resubmit. Repeated
or willful violations MAY result in loss of contribution privileges.

### Enforcement
Maintainers of the project are responsible for applying this standard. Because
intent and tool involvement cannot always be determined, enforcement focuses
on observable outcomes: whether the contributor understands and can defend the
contribution, whether it meets quality standards, and whether it is free of
known license problems. Contributions that fail these tests will be
handled the same way whether or not AI was involved.
