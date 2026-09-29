
# OAuth Client Attestation Profile for Agent Software Attributes

Working area for the individual Internet-Draft
`draft-mcguinness-oauth-agent-software-attestation`.

[Read the draft source](draft-mcguinness-oauth-agent-software-attestation.md).
After a local build, the [HTML draft](draft-mcguinness-oauth-agent-software-attestation.html)
and [plain-text draft](draft-mcguinness-oauth-agent-software-attestation.txt)
are available alongside the source.

The draft profiles [OAuth Client Attestation](https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/)
and reuses [EAT](https://www.rfc-editor.org/rfc/rfc9711.html) claims for
appraised runtime software, local model artifacts, and effective configuration.
It defines evidence association with the client key, freshness bounds,
software-change handling, and Receiver acceptance rules.

This is an initial working draft and a **partial EAT profile**. Deployment
Result Profiles must complete the cryptographic and appraisal choices.
It defines no new OAuth authentication method or JWT claim and has no
normative dependency on OAuth Mission or a stable instance identifier.
Open design questions appear in the draft's appendix.

## Build

Formatted HTML and text use the standard
[i-d-template tooling](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md):

```sh
make
```

The Makefile fetches the tooling into `lib` on first use. Generated documents,
reference caches, and local dependencies are ignored by Git. The existing
GitHub workflows build editor's copies; publication is a separate tagged
release action.

For an environment with `kramdown-rfc2629` and `xml2rfc` already installed,
the draft can also be built directly:

```sh
kramdown-rfc2629 draft-mcguinness-oauth-agent-software-attestation.md \
  > draft-mcguinness-oauth-agent-software-attestation.xml
xml2rfc --html --text draft-mcguinness-oauth-agent-software-attestation.xml
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the IETF contribution terms
and review workflow. Discussion and changes can be proposed through the
[repository](https://github.com/mcguinness/draft-mcguinness-oauth-agent-software-attestation).
