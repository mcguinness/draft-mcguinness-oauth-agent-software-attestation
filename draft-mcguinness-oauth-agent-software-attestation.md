---
title: "OAuth Client Attestation Profile for Agent Software Attributes"
abbrev: "Agent Software Attestation"
category: std
docname: draft-mcguinness-oauth-agent-software-attestation-latest
submissiontype: IETF
stand_alone: yes
ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - client attestation
 - agent software
 - EAT
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-agent-software-attestation"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-agent-software-attestation/draft-mcguinness-oauth-agent-software-attestation.html"
author:
 - fullname: Karl McGuinness
   organization: Independent
   email: public@karlmcguinness.com
normative:
  ATTEST: I-D.ietf-oauth-attestation-based-client-auth
  RFC6749:
  RFC6750:
  RFC9110:
  RFC7515:
  RFC7519:
  RFC7800:
  RFC8725:
  RFC9711:
informative:
  EAR: I-D.ietf-rats-ear
  INSTANCE-ID: I-D.mcguinness-oauth-client-instance-id
  RFC9126:
  RFC9334:
--- abstract

This specification defines an optional profile of OAuth 2.0
Attestation-Based Client Authentication for conveying appraised agent
software attributes. It reuses Entity Attestation Token claims to
describe runtime software, local model artifacts, and effective
configuration. The profile defines component roles, association with
the Client Instance Key, evidence freshness, handling of software
changes, and processing by an authorization server or resource server.
It is a partial EAT profile; deployments complete the cryptographic
and appraisal choices through full EAT profiles, called Result
Profiles. It defines no new OAuth authentication method or JWT claim.

--- middle

# Introduction {#intro}

An authorization server may require evidence that a client executes an
acceptable runtime build, loads a particular local model artifact, or
uses an acceptable configuration before issuing a token. A client
registration entry or a software name supplied by that client does not
establish these properties for the requesting Client Instance.

OAuth 2.0 Attestation-Based Client Authentication {{ATTEST}}
authenticates a Client Instance and allows its Client Attester to
convey further properties of the client's hardware and software. It
defines no claims for those properties, and the evidence behind them
is outside its scope ({{Section 14 of ATTEST}}). The Entity Attestation
Token (EAT) {{RFC9711}} defines claims for software names, versions,
and measurement results, and leaves their meaning in a particular use
to profiles ({{Section 6 of RFC9711}}). This specification is such a
profile for OAuth: it defines which software components a Client
Attestation describes, how their appraisal is associated with the
Client Instance Key, how old the underlying evidence can be, and how
an authorization server or resource server applies the result.

For example, a Receiver might require an appraised runtime build R,
local model artifact M, and effective configuration C, supported by
evidence no older than a configured interval. This specification
defines what those inputs mean and how the Receiver evaluates them.
The Receiver determines which Client Attesters, and which of their
appraisals, it accepts, and thereby which builds, artifacts, and
configurations.

The profile applies to clients that execute agent software. The same
rules can be used by other OAuth clients with equivalent software
requirements. It does not require a definition or registry of agent
types, model capabilities, or levels of autonomy.

## Scope and Specification Boundaries {#scope}

| Concern | Specification responsibility |
|---|---|
| Client authentication, proof methods, and HTTP carriage | {{ATTEST}} |
| Attestation claim syntax and EAT profiling | {{RFC9711}} |
| Software component roles, evidence association, and acceptance rules | This specification |
| Stable instance identity and continuity across key changes | An instance identification profile, such as {{INSTANCE-ID}} |
| Grants, scopes, actors, and delegation authority | The applicable OAuth authorization specification |
| Deployment approval and portable upstream provenance | Separate requirements outside this profile |

This profile covers appraised results presented directly in a Client
Attestation. {{Section 14 of ATTEST}} leaves Evidence, Endorsements,
and Reference Values out of scope; this profile constrains what the
Client Attester's appraisal of them establishes without defining their
formats. It does not define platform evidence acquisition, a hardware
attestation protocol, downstream access-token claims, or an
attestation chain for upstream participants.

The component roles defined here cover locally attestable execution.
Evidence that a client is configured to call a remote model service
does not establish which model handled a request at that service.
Remote inference provenance requires a separately specified mechanism.

# Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged-bcp14}

Client Attestation JWT, Client Attester, Client Instance, and Client
Instance Key are defined in {{Section 3 of ATTEST}}. This document
uses "Client Attestation" for a Client Attestation JWT and "proof" for
the Proof of Possession that accompanies it ({{Section 5 of ATTEST}}).
Evidence, Endorsements, Reference Values, Attestation Results, and
appraisal follow the RATS architecture {{RFC9334}}.

Receiver:
: An authorization server or resource server that directly validates
  the Client Attestation and its proof.

Component:
: The runtime, local model artifact set, or effective configuration
  described by one named submodule in the Client Attestation.

Execution Boundary:
: The environment, such as a process, container, or virtual machine,
  whose software the components describe and with which the Client
  Instance Key is associated. It is a Target Environment in the sense
  of {{Section 3.1 of RFC9334}}.

Appraisal Contract:
: The Client Attester's appraisal behind one measurement-system
  identifier ({{results}}): the evidence sources and Endorsements it
  accepts, the Reference Values it compares against, and the
  measurement coverage and evidence strength of each comparison. In
  RATS terms, it is an Appraisal Policy for Evidence together with its
  Reference Values. Receivers rely on it through their trust in the
  Client Attester and do not consume its Reference Values. It is not a
  protocol object or wire parameter.

Result Profile:
: A full EAT profile that incorporates this specification and resolves
  the remaining choices listed in {{profile-completion}}. It describes
  the Client Attester's output, independently of the formats used for
  underlying platform evidence.

# Architecture {#architecture}

The Client Attester appraises evidence about the Client Instance and
issues a Client Attestation containing the resulting attributes. The
Client Instance presents that attestation with the proof required by
{{ATTEST}}. The Receiver validates both artifacts before using the
attributes in an authorization decision.

~~~
  Client Instance            Client Attester              Receiver
        |                           |                         |
        | evidence, instance key    |                         |
        |-------------------------->|                         |
        |                           | appraise evidence and   |
        |                           | key association         |
        | Client Attestation        |                         |
        |<--------------------------|                         |
        |                                                     |
        | OAuth request, Client Attestation, and proof        |
        |---------------------------------------------------->|
        |                                validate, then apply |
        |                                     software policy |
~~~
{: title="Profile Data Flow"}

In RATS terms ({{Section 14 of ATTEST}}), the Client Instance is the
Attester, the Client Attester is the Verifier, the Client Attestation
is an Attestation Result, and the Receiver is the Relying Party; the
exchange follows the Passport Model ({{Section 5.1 of RFC9334}}). The
Client Attester MUST issue claims under this profile only after
completing the
appraisal and association checks in {{attester-processing}}. Forwarding
an unverified client declaration inside a signed JWT does not satisfy
those checks.

The Receiver MUST establish trust in the Client Attester for the
specific components and Appraisal Contracts it accepts. Trust in a
Client Attester for client authentication alone does not establish its
competence or authority to appraise a model artifact or configuration.

# Profile Selection and Completion {#configuration}

Three separately maintained inputs govern a decision under this
profile, and each has one owner:

Result Profile:
: Written by its author for every Client Attester and Receiver that
  implements it, and identified by the `eat_profile` value. It defines
  the token format, algorithms, key identification, component and
  claim requirements, measurement schemes and result identifiers, key
  association, the bounds B, L, and S ({{freshness}}), and any
  state-at-use mechanism ({{profile-completion}}).

Appraisal Contract:
: Maintained by a Client Attester and identified by a
  measurement-system identifier. It defines evidence sources,
  Endorsements, Reference Values, measurement coverage, and evidence
  strength.

Receiver configuration:
: Maintained by a Receiver for each client and operation. It defines the
  accepted Result Profiles, the Client Attesters it trusts and the
  components and Appraisal Contracts for which it trusts each, the
  required checks, the limits A and W ({{freshness}}), and whether
  state-at-use enforcement is required. In RATS terms, it is an
  Appraisal Policy for Attestation Results.

Use of this profile MUST be selected through trusted Receiver
configuration for the relevant client and operation. Selection MUST
identify the acceptable Result Profiles, Client Attesters, required
components and checks, and freshness limits. This is how a Receiver
determines that the profile applies, which {{Section 13 of ATTEST}}
requires a profile to define; the selection can be established out of
band.

A Receiver MUST NOT infer that this profile is required merely from
an unverified token, and MUST NOT allow the client to disable a
configured requirement by omitting the profile or its claims. When the
profile is required, ordinary client authentication without the
required attributes is insufficient for that operation.

At least one check on the `runtime` component MUST be required. Each
required component MUST have one or more required checks; component
presence alone is insufficient.

The `eat_profile` claim MUST identify an accepted Result Profile. A
Receiver MUST match the identifier to trusted configuration, validate
the Client Attester using that configuration, and apply the complete
profile.
A token-provided identifier MUST NOT create a new trust relationship.
Profile identifiers and measurement-system identifiers MUST NOT be
automatically dereferenced as part of request processing.

## Completing the Partial EAT Profile {#profile-completion}

This document is a partial EAT profile ({{Section 6.2 of RFC9711}}).
In particular, it leaves the algorithm suite, key association, and
freshness bounds to a Result Profile. Because EAT does not allow
`eat_profile` to identify a partial profile, the claim always
identifies a Result Profile and never this document. A Result Profile
MUST identify the revision of this specification that it incorporates.

Each Result Profile MUST specify the following:

* An identifier, which is a URI or OID ({{Section 4.3.2 of RFC9711}});
  the allowed digital signature algorithms, all of which its Receivers
  implement; and key identification and trusted key resolution.
* The supported software naming and version schemes; the measurement
  schemes, including how measurement-system identifiers are assigned
  and which result identifiers each scheme defines and what they mean;
  and the kinds of evidence sources and Endorsements on which
  Appraisal Contracts can rely. Individual Appraisal Contracts and
  their Reference Values are not part of a Result Profile.
* How evidence is associated with the Client Instance Key, including
  which Execution Boundary is measured and which software can use the
  corresponding private key.
* The evidence age bound B, maximum lifetime L, and clock uncertainty
  bound S defined in {{freshness}}. The acceptance limits A and W are
  Receiver configuration, not Result Profile values.
* Required, optional, and prohibited components and claims, size
  limits, and any additional EAT claims. If `measurements` or
  `manifests` are carried, their formats and processing requirements
  MUST be specified.
* Any enforceable state-at-use mechanism it offers, and its failure
  behavior ({{state-changes}}). Without one, acceptance means a
  bounded-age observation.

A Result Profile MUST resolve all applicable choices in
{{Section 6 of RFC9711}}. The result is a JSON-encoded JWT with a JWS
signature ({{representation}}); CBOR encoding choices, MAC protection,
Unsecured JWTs, encryption, and detached EAT bundles do not apply. The
freshness mechanism ({{Section 6.3.11 of RFC9711}}) is the one in
{{freshness}}. This profile does not use the `eat_nonce` claim; replay
of a presentation is addressed by the proof ({{ATTEST}}). The accepted
platform evidence format can differ from the result format.

A new set of Reference Values is published as a new Appraisal
Contract under a new measurement-system identifier and does not
require a new Result Profile. Different Receivers can accept different
Appraisal Contracts under the same Result Profile. A change to the
meaning or freshness guarantees of a Result Profile requires a new
profile identifier. A mutable URL whose contents change without a new
identifier is insufficient to identify those guarantees.

# Client Attestation Representation {#representation}

The Client Attestation MUST be a JWT {{RFC7519}} protected with a
digital signature using JWS Compact Serialization {{RFC7515}}; the MAC
protection that {{Section 4 of ATTEST}} also permits is not used. The
`typ` JOSE header parameter remains `oauth-client-attestation+jwt`.
The `sub`, `exp`, and `cnf` claims retain their meanings and
requirements from {{Section 4 of ATTEST}}, including the `jwk` member
of the `cnf` claim {{RFC7800}}. This profile additionally requires the
`iat`, `eat_profile`, and `submods` claims. The `iat` and `exp` values
MUST be integer NumericDate values, and `exp` MUST be greater than
`iat`.

The `sub` value continues to identify the OAuth client. It MUST NOT
be repurposed as a model name, deployment identifier, or principal.
The proof methods, endpoint authentication methods, HTTP fields, and
challenge processing remain those of {{ATTEST}}. Selecting this profile
does not select a different proof method.

The following table lists where this profile narrows {{ATTEST}} and
{{RFC9711}}.

| Base specification | This profile | Section |
|---|---|---|
| `iat` optional ({{Section 4 of ATTEST}}) | `iat` present, as an integer | {{representation}} |
| Signature or MAC ({{Section 4 of ATTEST}}) | Digital signature only | {{representation}} |
| Freshness by local policy ({{Section 7.1 of ATTEST}}) | Evidence age and elapsed time limits | {{freshness}} |
| Submodule semantics left to profiles ({{Section 4.2.18 of RFC9711}}) | `runtime`, `model`, and `configuration` roles as submodule Claims-Sets | {{components}} |
| `measres` group names and result identifiers of any form ({{Section 4.2.17 of RFC9711}}) | Absolute-URI measurement-system identifiers; nonempty text result identifiers | {{results}} |

## Components {#components}

Components MUST be represented as submodule Claims-Sets
({{Section 4.2.18.1 of RFC9711}}) in the `submods` claim. The `runtime`
component is REQUIRED. The `model` and `configuration` components are
OPTIONAL unless the Result Profile or Receiver configuration requires
them. Each of these names occurs at most once. Nested submodules,
nested tokens, and detached submodule digests MUST NOT be used for
these components. A `runtime`, `model`, or `configuration` component
that violates this section or {{results}} is invalid and satisfies no
check.

| Submodule name | Meaning |
|---|---|
| `runtime` | The executing client runtime within the appraised Execution Boundary |
| `model` | The local model artifact set loaded by that runtime, including the dependencies covered by the Appraisal Contract |
| `configuration` | The effective configuration applied to that runtime and local model, within the declared measurement coverage |

A component's measurement coverage MUST be explicit in the Appraisal
Contract. For example, an Appraisal Contract for the `model` component
can cover weights, tokenizer files, and adapters, and one for the
`configuration` component can cover sandbox settings and tool
allowlists. Omitted dependencies and dynamically supplied inputs are
outside the claim unless the Appraisal Contract explicitly covers them.

A component MAY contain `swname` and `swversion`, and a Result Profile
can require them, for example to support name or version comparisons in
Receiver configuration. `swversion` uses its array encoding; any other
form is malformed. In every component, `swversion` appears only with
`swname` ({{Section 4.2.7 of RFC9711}}). Because `swname` is free text
({{Section 4.2.6 of RFC9711}}), it identifies software less precisely
than the measurement results. A Receiver MUST NOT infer artifact
integrity from a name or version alone. Name and version comparisons
MUST follow the configured scheme; a Receiver MUST NOT invent version
ordering or treat a moving alias as an immutable artifact identifier.

Other submodule names do not acquire a component role from this
specification, and the restrictions in this section do not apply to
them. They MUST NOT substitute for a required component. Unknown
claims are ignored as required by {{Section 4 of ATTEST}}; an unknown
or uninterpretable claim cannot satisfy a required check.

## Appraised Measurement Results {#results}

Each component MUST contain `measres` using the EAT representation in
{{Section 4.2.17 of RFC9711}}. This profile constrains a measurement
group's measurement-system name to an absolute URI, the
measurement-system identifier, that identifies one immutable Appraisal
Contract. Result identifiers MUST be nonempty text strings, with
meanings defined by the measurement scheme in the Result Profile.
Receivers compare result identifiers as text and never base64url-decode
them, even though EAT also permits binary result identifiers.

A Receiver MUST evaluate each required check using the tuple of
component role, measurement-system identifier, and result identifier.
It MUST NOT accept a `success` result for a different tuple as a
substitute. Identifier comparison is exact and case sensitive, with
no URI normalization. Within a component, a measurement-system
identifier MUST occur at most once, and result identifiers within a
group MUST be unique. Duplicate identifiers make that component
invalid for this profile.

Receiver configuration can accept more than one measurement-system
identifier for the same component role and result identifier, and a
Client Attester can report results for one component under several
Appraisal Contracts. Together these allow a move to a new Appraisal
Contract without requiring every Receiver to change at once; a result
under any accepted identifier satisfies the check.

For a required check, only `success` satisfies the measurement
comparison. `fail`, `not-run`, `absent`, and a missing result all fail
the requirement. As in {{Section 4.2.17 of RFC9711}}, each result
reports a comparison with Reference Values, not an overall appraisal
verdict; association with the Client Instance Key is asserted by the
presence of the component ({{attester-processing}}). Successful
comparison remains subject to the association, freshness, trust, and
authorization checks in this specification.

An Appraisal Contract MUST define the Reference Values, measurement
coverage, and evidence strength needed for each successful comparison.
A Client Attester MUST NOT report a successful measured-artifact check
solely because the client supplies an approved name or version.
Installed files alone MUST NOT be reported as evidence that those
files are loaded by the requesting runtime.

Raw evidence need not be disclosed to the Receiver. A Result Profile
can allow EAT `measurements` or `manifests` claims when justified,
using their EAT encodings and the formats it specifies
({{profile-completion}}). These claims do not replace the required
appraisal results or association checks. A software manifest alone
does not establish execution.

EAT Attestation Results {{EAR}} define another attestation result
format. This profile does not use it, because EAR requires a fixed
top-level `eat_profile` value, which would displace the Result Profile
identifier on which Receivers rely ({{configuration}}), and because it
depends on Internet-Drafts not yet published as RFCs.

# Client Attester Processing {#attester-processing}

Before issuing a Client Attestation conforming to this profile, the
Client Attester MUST:

1. Authenticate the evidence source and validate evidence integrity,
   freshness, replay protections, and Endorsements under the relevant
   Appraisal Contract.
2. Establish that each reported component belongs to the appraised
   Execution Boundary associated with the Client Instance Key in the
   `cnf` claim. Evidence about another workload on the same host is
   insufficient.
3. Verify that the measured component state supports each reported
   name, version, and comparison result, including any required
   relationship between an artifact on disk and the loaded artifact.
4. Complete the appraisal and ensure that all evidence used in the
   result meets the evidence age bound B in {{freshness}}.
5. Issue the result with an accurate `iat`, bounded `exp`, and the
   applicable Result Profile identifier.

Association can be established by platform evidence that binds the
public key to a measured Execution Boundary, or by another
authenticated mechanism explicitly accepted in the Result Profile. A
bare public key, client-supplied process identifier, or unrelated
proof of key possession MUST NOT be treated as evidence of that
association.

Submodules inherit nothing from the enclosing token
({{Section 4.2.18 of RFC9711}}), so EAT itself does not relate a
component to the `cnf` claim. By signing a Client Attestation that
contains a `runtime`, `model`, or `configuration` component, the
Client Attester asserts that it completed the association check in
step 2 for that component.

Names, versions, and results in the attested component claims MUST
reflect the appraised evidence. Self-declared inventory data MUST NOT
be placed in these claims as if it had been appraised.

# Freshness and Software Changes {#freshness}

A fresh proof establishes possession of the Client Instance Key for
the request. It does not refresh the software evidence or the Client
Attester's appraisal.

{{Section 4.3.1 of RFC9711}} defines `iat` as the time at which claims
are collected and the token is signed, and notes that the data behind
some claims can be older. Rather than adding per-claim timestamps, this
profile bounds that difference. A Result Profile MUST define a finite
bound B, in seconds, on the age at `iat` of the oldest observation of
component state on which the result relies. For a measurement taken
when an artifact was loaded, that is the time of loading, not the time
the Evidence was conveyed or appraised. A measurement scheme that
observes state only at load time can therefore support results only
within B of loading; later results require a scheme that observes the
running state again. The bound MUST include the uncertainty of
evidence timestamps and any delay before appraisal.
Each issued result MUST reflect a completed appraisal. A Client
Attester that re-signs a cached appraisal MUST NOT reset the age of
its underlying evidence; B still applies to the original observation.

A Result Profile MUST also define a finite maximum lifetime L and a
clock uncertainty bound S between the Client Attester and Receiver.
The Client Attester MUST ensure `exp - iat <= L`. Receiver
configuration MUST define a maximum age A for the evidence it accepts
and a maximum elapsed time W since issuance. W limits how long a
Receiver accepts reuse of one Client Attestation
({{Section 10.2 of ATTEST}}); A limits the age of the evidence behind
it and matters when a Receiver accepts Result Profiles with different
values of B. These are configuration values, not JWT claims, and are
expressed in seconds. All five durations MUST be finite and
nonnegative; L MUST be positive.

For Receiver time N, the Receiver MUST reject a Client Attestation
when `iat > N + S`, `exp - iat > L`, or either of the following holds:

~~~
  N - iat + S > W
  B + N - iat + S > A
~~~

Once `iat > N + S` is excluded, `N - iat + S` is nonnegative and
bounds the time elapsed since issuance, and `B + N - iat + S` bounds
the age of the oldest observation. A Client Attestation received at
its `iat` passes these checks only if W is at least S and A is at
least B + S.

In the notation of {{Appendix A of RFC9334}}, `iat` is time(RG_v),
`exp` is time(RX_v), N is time(RA_r), and the observation that B
bounds is time(VG_a). B bounds time(RG_v) - time(VG_a), W bounds
time(RA_r) - time(RG_v), and A bounds time(RA_r) - time(VG_a); S
accounts for the difference between the Verifier and Relying Party
clocks.

The expiration check of {{Section 4 of ATTEST}} also applies, with S
as its allowable clock skew. The Receiver MUST take B, L, and S from
the accepted Result Profile and A and W from its configuration, and
MUST NOT relax any of them based on a token's requested values. Clock
uncertainty cannot be added again as an extra grace period to these
checks.

This conservative calculation allows the Receiver to bound evidence
age without learning every measurement timestamp. Deployments needing
per-component timestamps or challenge-bound evidence can define a
Result Profile that adds an appropriate evidence mechanism. A proof
challenge ({{Section 6 of ATTEST}}) alone does not provide that
mechanism.

## Changes After Appraisal {#state-changes}

An observation is evidence about a state at a time. A valid signature
and an unexpired result do not ensure that software remains unchanged.
Receivers MUST interpret the component results in a Client
Attestation as a bounded-age observation unless the Result Profile
defines an enforceable state-at-use mechanism and the Receiver
configuration requires it.

After a change to an appraised runtime, model, configuration, or
covered dependency, the Client Instance MUST NOT present a Client
Attestation issued before the change, and the Client Attester MUST NOT
issue results for the changed state from evidence collected before the
change. Retaining the same instance identifier or Client Instance Key
does not preserve software state.

A requirement that software remain acceptable at request time needs
an additional enforcement mechanism. Examples include restricting key
use to the measured state or checking authenticated invalidation
information before accepting a request. The Result Profile MUST
identify the mechanism and failure behavior. A promise by the Client
Instance to obtain a new attestation, or a short expiration alone,
does not establish such enforcement against a compromised Client
Instance.

This specification defines no revocation distribution protocol. An
old JWT does not become cryptographically invalid merely because a
new appraisal was issued. A Result Profile MUST NOT claim immediate
invalidation unless its Receivers can enforce it.

# Receiver Processing {#receiver-processing}

When this profile is required for an operation, the Receiver MUST:

1. Select the applicable trusted configuration and accepted Result
   Profiles for the client and operation.
2. Validate the Client Attestation and proof under {{ATTEST}},
   including client identity, signatures, key association in the proof,
   audience, challenge, and replay checks applicable to the selected
   proof method. JWT validation follows {{RFC8725}}.
3. Validate the Result Profile identifier, trust in the Client
   Attester for the required components and Appraisal Contracts,
   representation, and required claims. A signed profile identifier
   alone is insufficient to establish that trust.
4. Enforce the freshness rules in {{freshness}} and any configured
   state-at-use enforcement in {{state-changes}}.
5. Locate all required components and measurement-result tuples and
   require each configured check, including any configured name or
   version comparison ({{components}}), to succeed. An unknown
   Appraisal Contract or unsupported result meaning cannot satisfy a
   required check.
6. Apply the operation's authorization policy using the validated
   attributes. The grant, scope, actor, resource, and other applicable
   authorization checks remain necessary.

A Receiver MUST NOT continue with a less demanding software policy
because a required component, profile, or result is absent or
unacceptable. A client that can authenticate successfully may still
fail the software requirement and be refused the operation.

## Error Handling {#errors}

Receivers MUST use the error handling of {{Section 7.4 of ATTEST}}
and the applicable endpoint specification, including {{RFC6749}} for
authorization server endpoints and {{RFC6750}} for bearer-protected
resources. Failures under this profile fall into three classes:

Not fresh enough:
: An expired Client Attestation, or a failure of the W or A check in
  {{freshness}}. {{Section 7.4 of ATTEST}} requires the
  `use_fresh_attestation` error code for this condition. A new proof
  over the same Client Attestation cannot resolve it.

Attestation validation failure:
: An invalid signature, an untrusted Client Attester or unaccepted
  Result Profile, a violation of {{representation}} outside the
  components, `iat > N + S`, `exp - iat > L`, or an invalid proof.
  These are handled as an
  attestation that could not be successfully verified
  ({{Section 7.4 of ATTEST}}), not as a freshness failure, so that a
  client does not repeatedly obtain new Client Attestations that
  cannot succeed.

Software policy failure:
: A valid Client Attestation whose components or results do not
  satisfy a required check, including a required component that is
  missing or invalid ({{components}}). This is not a request for a new
  proof challenge. An authorization server MUST respond with the
  `unauthorized_client` error code ({{Section 5.2 of RFC6749}}), which
  also applies at endpoints that reuse that error format, such as the
  pushed authorization request endpoint ({{Section 2.3 of RFC9126}}).
  A resource server MUST respond with HTTP status code 403 (Forbidden)
  ({{Section 15.5.4 of RFC9110}}). None of the error codes in
  {{Section 3.1 of RFC6750}} describes this condition, so the response
  carries no `error` attribute; in particular, this profile does not
  broaden the meaning of `insufficient_scope` to cover software
  failures. This specification introduces no new error code.

Error descriptions SHOULD avoid exposing detailed software inventory,
accepted measurement-system identifiers, or Client Attester
configuration.

## OAuth Composition {#composition}

When the operation is token issuance, the authorization server
evaluates the requesting client's attributes. Attributes obtained from
an upstream token or another participant MUST NOT substitute for that
client's required appraisal.

When policy requires this profile for a refresh or token exchange, the
request MUST satisfy the same configured software checks as any other
covered request. This profile does not change the refresh token
binding in {{Section 10.3 of ATTEST}}, the key rotation rule in
{{Section 10.6 of ATTEST}}, or the corresponding rules of a separately
selected profile.

An authorization server's acceptance of an attestation does not
automatically convey the attributes or their freshness to a downstream
resource server. This specification defines no access-token or
introspection representation. A Receiver requiring direct attestation
uses {{ATTEST}}; a deployment requiring downstream propagation needs
a separately specified, authenticated representation and trust model.

A key used to sender-constrain access tokens, such as a DPoP key, can
differ from the Client Instance Key; the two are required to be the
same only in the combined mode of {{Section 5.2 of ATTEST}}. Any
association required
for downstream attribution follows the applicable token and instance
context specifications.

# Examples {#examples}

These examples illustrate profile composition and processing. The
identifiers are illustrative, and the decoded JWT is not a signed
interoperability test vector.

## Example Deployment {#example-contract}

Assume a Result Profile identified by
`https://attester.example/profiles/agent-software/v1`. It incorporates
this specification and selects JWS with ES256, `kid` resolution
against preconfigured Client Attester keys, and inline JSON results
only. Receivers implementing that profile support ES256. It defines
the result identifier `loaded-artifact` for the `runtime` and `model`
components and `effective-config` for the `configuration` component.
Its association mechanism binds the measured Execution Boundary to the
Client Instance Key. Software names are exact-match labels, and
versions use the single-string EAT array form with no implicit
ordering. It sets B to 30 seconds, L to 120 seconds, and S to 5
seconds, and it supplies bounded-age observations with no claim of
immediate change detection. The Client Instance uses the Client
Attestation PoP JWT method ({{Section 5.1 of ATTEST}}).

The Client Attester publishes an Appraisal Contract under each
measurement-system identifier used below. For example, the one
identified by `https://attester.example/reference/model/42` covers the
loaded model weights, tokenizer, and adapters.

The Receiver's configuration for the operation requires `success` for
all three checks under those identifiers and sets W to 60 seconds and
A to 75 seconds. These are example values, not recommended defaults.

## Decoded Client Attestation {#example-attestation}

The following example shows the decoded protected header and payload
of a Client Attestation under that Result Profile.

Protected header:

~~~ json
{
  "typ": "oauth-client-attestation+jwt",
  "alg": "ES256",
  "kid": "attester-key-1"
}
~~~

Payload:

~~~ json
{
  "sub": "https://client.example/agent",
  "iat": 1790632800,
  "exp": 1790632920,
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "VcKVNBZ4IaBAYW3jxM4w3TJFVA7myeUGQyGt-g_yvpQ",
      "y": "f-E-hYE3TAWKwhVv9pej9NABs9SX9XsNO80x57jFTyU"
    }
  },
  "eat_profile":
    "https://attester.example/profiles/agent-software/v1",
  "submods": {
    "runtime": {
      "swname": "example-agent-runtime",
      "swversion": ["2.4.1"],
      "measres": [
        ["https://attester.example/reference/runtime/17",
          [["loaded-artifact", "success"]]]
      ]
    },
    "model": {
      "swname": "example-local-model",
      "swversion": ["2026.09"],
      "measres": [
        ["https://attester.example/reference/model/42",
          [["loaded-artifact", "success"]]]
      ]
    },
    "configuration": {
      "measres": [
        ["https://attester.example/reference/config/8",
          [["effective-config", "success"]]]
      ]
    }
  }
}
~~~

The `jwk` member of the `cnf` claim is the Client Instance's public
key, not the Client Attester's signing key. The Receiver obtains the
latter from trusted configuration and validates the JWT signature. The
Client Instance supplies the corresponding proof separately.

At Receiver time `iat + 40`, the elapsed-time bound is `40 + 5 = 45`
seconds, which meets W, and the evidence age bound is `30 + 45 = 75`
seconds, which meets A. At `iat + 45`, the evidence age bound is 80
seconds, which fails A even though the elapsed-time bound of 50
seconds meets W and the attestation is unexpired. From `iat + 56`, the
W check also fails. Creating a new proof does not change any of these
results.

## Acceptance and Refusal Cases {#cases}

| Case | Required outcome |
|---|---|
| Trusted Client Attester, associated key, acceptable results, and fresh evidence | Software checks pass; ordinary authorization checks still apply |
| Evidence from one instance offered for another instance's key | Client Attester refuses to assert the association |
| A valid Client Attestation for one key presented with a proof from another key | Receiver rejects the proof |
| Approved model name with no required appraisal result | Receiver refuses the operation |
| Required result is `fail`, `not-run`, `absent`, or missing | Receiver refuses the operation |
| `success` under a measurement-system identifier the Receiver does not accept | Receiver refuses the operation |
| Duplicate result identifier with conflicting statuses | Component satisfies no check |
| Fresh proof over an attestation that fails the A or W check | Receiver requires a fresh attestation |
| Token is newly signed using evidence older than B | Client Attester violates the profile; the Receiver cannot detect this from the token |
| Loaded model changed after appraisal | Client Instance stops presenting the earlier Client Attestation; immediate Receiver detection needs the configured enforcement mechanism |
| Client reports the name of a remote inference service | Does not satisfy a required local model appraisal |
| Client omits a required profile and uses another authentication method | Receiver refuses the covered operation |
| Upstream participant has acceptable attributes but requesting client does not | Upstream attributes do not satisfy the requester's requirement |

# Security Considerations {#security}

The security considerations of {{ATTEST}}, {{RFC9711}}, and
{{RFC8725}} apply.

## Evidence Substitution and Key Sharing

An attacker can combine valid evidence from a compliant workload with
a key usable by a different workload. The association requirements in
{{attester-processing}} address this substitution. Host identity,
co-location, or an administrative label alone is insufficient when the
policy concerns one process or Execution Boundary.

If several workloads can use the same private key, successful proof
does not distinguish which workload made the request. The Result
Profile states which software can use the private key
({{profile-completion}}), and a Receiver MUST NOT infer isolation finer
than that boundary. A software-only association mechanism can be
acceptable under one Result Profile while being inadequate for a
Receiver that requires hardware isolation.

## State Changes and Incomplete Coverage

Files may be replaced after measurement, models can load adapters at
runtime, and configuration can change without replacing the executable.
The Appraisal Contract needs to account for these changes if they
affect the policy.
Hashes of selected files do not establish completeness of the measured
state. The limits in {{state-changes}} remain relevant even when every
signature verifies.

An acceptable model artifact does not guarantee safe output, freedom
from prompt injection, correct tool use, or compliance with a user's
intent. Appraised software attributes MUST NOT be interpreted as an
authorization grant or as proof of those behavioral properties.

## Client Attester and Policy Compromise

A Client Attester can issue false results or misstate evidence age.
The Receiver's confidence depends on the Client Attester's validation
process, key protection, and operational controls. Signature
validation cannot independently establish that B was respected.
Investigating a false result depends on the Client Attester retaining
which evidence, Appraisal Contract, and association checks supported
it, subject to its retention and privacy policy. Receivers SHOULD
support withdrawal of trust in Client Attester keys and Appraisal
Contracts when those assumptions fail.

Measurement-system identifiers MUST identify fixed semantics. A
Receiver cannot verify that an identifier is immutable; it relies on
the Client Attester. Reusing an identifier after weakening its checks
enables an old acceptance policy to authorize a different software
state. Policy updates also need to address rollback to previously
acceptable but vulnerable versions.

## Parsing and Resource Limits

Receivers MUST reject duplicate JSON member names in the protected
header and claims sets. They MUST enforce the configured token size
and component limits before performing expensive processing. They MUST
apply algorithm restrictions from trusted configuration rather than
accepting an algorithm solely because the JWT names it.

Automatic retrieval of token-provided profile, measurement-system, or
key URLs can enable server-side request forgery or untrusted
trust-anchor selection. {{configuration}} forbids dereferencing
profile and measurement-system identifiers during request processing.
Any retrieval of Client Attester keys, such as through the `jku`
mechanism in {{Section 10.8 of ATTEST}}, needs to be limited to
locations that trusted configuration establishes independently of the
presented token.

# Privacy Considerations {#privacy}

The privacy considerations of {{Section 11 of ATTEST}} and
{{Section 8 of RFC9711}} apply.

Software names, versions, configuration details, and unusual
combinations of artifacts can identify deployments and reveal
vulnerabilities or proprietary information. Result Profiles and
Receiver configuration SHOULD require only the components and results
necessary for the decisions they support, and Result Profiles can
prohibit claims, such as `ueid`, `sueids`, and `location`, that those
decisions do not need. Client Attesters SHOULD prefer appraised
results over raw evidence where that satisfies the requirement.

Client Attesters MUST NOT put credentials, prompts, user content, or
other secrets in software names, versions, or result identifiers.
Configuration measurements can also expose low-entropy secrets through
guessing attacks; omitting plaintext does not remove that risk.

This profile introduces no permanent device identifier. Its software
attributes can still contribute to correlation, including when combined
with the Client Instance Key. Using a different Client Instance Key for
each Receiver, as {{Section 11.1 of ATTEST}} recommends, does not
prevent correlation through a distinctive combination of software
attributes. Retention and disclosure policies SHOULD account for that
possibility.

{{ATTEST}} allows a Client Instance to obtain a Client Attestation
without revealing to the Client Attester which authorization servers
or resource servers it will contact. A Client Instance that requests a
Result Profile or Appraisal Contract specific to one Receiver reveals
that Receiver to the Client Attester; deployments SHOULD consider that
tradeoff when choosing shared or Receiver-specific Result Profiles and
Appraisal Contracts.

# IANA Considerations {#iana}

This document has no IANA actions. It reuses existing JWT claims and
OAuth mechanisms. The component names are values within `submods`,
not new JWT claim names. Result Profile and measurement-system URIs
are assigned by their respective authorities; this document does not
establish a registry for them.

--- back

# Design Questions {#design-questions}

This section records questions for review and is to be removed before
publication as an RFC.

* Which independently implemented agent deployments need the same
  result semantics? A shared concrete Result Profile should be driven
  by those deployments and their evidence sources.
* Is the conservative evidence age bound sufficient, or is a separate
  interoperable per-component observation-time mechanism justified?
* Do deployments need multiple local models or independently appraised
  configuration layers in one result? The initial named roles avoid
  defining an unmotivated component inventory schema.
* Is portable artifact identification required in addition to comparison
  against immutable Appraisal Contracts? If so, which existing EAT
  measurement or manifest format meets that requirement?
* Is a common state-at-use enforcement mechanism feasible across the
  relevant platforms, or should such guarantees remain specific to
  each Result Profile?
* Should an error code for software policy failure at a resource
  server be registered, as {{ATTEST}} registers codes for attestation
  failures? {{Section 3.1 of RFC6750}} otherwise expects one of its
  error codes in an error response.

# Document History

## Initial Working Draft

* Define an OAuth Client Attestation profile and partial EAT profile
  for appraised runtime, local model, and configuration attributes.
* Reuse EAT component and measurement-result claims with explicit
  evidence association, freshness, and software-change rules.
* Keep instance identity, upstream provenance, authorization, and
  deployment approval outside this profile.
