---
title: |-
  SLSA: Safeguarding artifact integrity across any software
  supply chain
date: 2026-01-21 10:50:00 +13:00
---

Kia ora,

Lately I've been working on [SLSA level 3](https://handbook.gitlab.com/handbook/engineering/architecture/design-documents/slsa_level_3/)
implementation while at my day job. If you know me personally, you'll know that
I do not really burn with passion for security standards such as SLSA, but I
have learned lot about it and have been wanting to write more blog posts. I thought it could be found by people who are working on it who are looking for a
primer on what it is and what it is useful for. Note that this is my personal
opinion and does not reflect the opinion of my employer.

I could grab the definitions in the SLSA spec and embed them into the blog
here, but I think that [you can read it yourself, if you really want
to.](https://slsa.dev/). I also think that you, like most people who are sane
of mind, absolutely do not want to. Not under any circumstances.

SLSA markets itself as: "Safeguarding artifact integrity across any software
supply chain". The main word to define here is "safeguarding".

For context, in this year 2026 of our lord, blue teams left and right are being
popped right and centre by insecure dependencies.  We've seen this quite a bit
and will continue to see it more and more. An example of this is: [Widespread
supply chain compromise impacting npm
ecosystem](https://www.ncsc.govt.nz/alerts/widespread-supply-chain-compromise-impacting-npm-ecosystem/).

SLSA can fix this for you, provided you do all the hard work for yourself.

SLSA has several levels of compliance, which map very roughly to:

* Level 1: You're producing metadata for your binaries. 
* Level 2: The provenance you're producing is signed.
* Level 3: Not only is the provenance you've produced signed, but it meets
  much stronger security guarantees. An example of this is: "the machine that produces the binary cannot access the private key required to perform the signature", or "[Unforgeability](https://slsa.dev/spec/v1.0/requirements#provenance-unforgeable)".

By and large, SLSA is all about metadata. An example of this metadata is
"Provenance" statements. Provenance statements, as defined by SLSA, is metadata about a specific build process. It allows a consumer of said
metadata to make risk-based decisions on specific binaries it is inspecting. 

For example:

* Has this binary been produced by someone I trust? This can be confirmed by signature verification or OIDC integration. By the latter, we can confirm whether a binary was signed by a specific GitHub repository.
* What parameters was this build configured with?
* What dependencies are associated with this build?

The full spec of this metadata can be found in the SLSA [build provenance
page](https://slsa.dev/spec/v1.2/build-provenance), although other types of
metadata can also be embedded. **The long and short
of it is that SLSA is whatever you want it to be.** By putting the data you want
in the provenance statement, you can then retrieve this data before executing a binary.

In this way, SLSA is somewhat similar to GPG signing a specific binary with a
key, but it does have additional infrastructure that can be helpful in managing
the cryptographic aspects that make GPG signatures so challenging. I mentioned
above "OIDC integration". By leveraging [OIDC](https://openid.net/developers/how-connect-works/), it is possible to have a
cryptographic signature that can be verified without providing the machine that
builds your binary the private keys associated with the signing.

How can this magic possibly work? I can tell you. A OIDC flow will have three participants, when looking to sign an attestation:

- End user. This is who wants to sign a binary, i.e. you.
- Relying party. This is [Fulcio](https://github.com/sigstore/fulcio), a tool that aids these type of signatures. Sigstore provide a “public good” server that you can use if hosting your own instance of Fulcio seems like too much.
- OpenID Provider. This can be GitLab, GitHub, Google etc.

At a high-level Fulcio will:

* Accept the JWT token you provide, perform basic validation on it.
* Send the token to the OpenID provider (e.g. GitLab).
* GitLab will verify the token and return a Claim.
* Fulcio will use the claim to perform the signature. The signature is a basically a statement: the end user authenticated as Claim at this time using this specific OIDC. This is referred to as the “identity” of the signer.
* Send the certificate to Rekor. Rekor is a certificate transparency type thing, so that if there is malicious activity it can be detected.

The threat model page is very interesting, and shows us what we can expect in terms of guarantees from Sigstore's way of doing things:

> Under normal operation (no Sigstore compromise), verifying a “keyless” signature from user@example.com using the ExampleIdP identity provider at a given timestamp guarantees that the signature was created by a signer who successfully authenticated to Sigstore using that identity at that time.

In order to verify the signature, we need to communicate with verifiers the identity of the signer. People looking to verify the identity of the signer could run the following command:

```
cosign verify <image URI> --certificate-identity=https://gitlab.com/your-repo/* --certificate-oidc-issuer=gitlab
```

Auth is interesting in itself. More info: [Use Sigstore for keyless signing and verification](https://docs.gitlab.com/ci/yaml/signing_examples/) and [Sigstore CI Quickstart - Sigstore](https://docs.sigstore.dev/quickstart/quickstart-ci/).

# In summary

SLSA is a common standard for specifying metadata, which is accompanied by
infrastructure for key management. Actually solving the security issues depends
on the implementation by each end user of the specification.

You could imagine a world in which this metadata can be used to aid detection
of known-bad package files: There is a centralised repo somewhere that has a
list of all compromised packages, and if the metadata analysis says a package
is being used that is present in that list, a deployment may fail.

In practice, this is not that simple. For starters, any false positives would
be totally unacceptable in most companies, as broken pipelines are a
productivity killer. Most people would be OK with halting deployments to
prevent remote code execution, but there may be vulnerabilities in a package
that are perhaps not applicable to your current setup, or are not severe enough
to warrant a complete halt of pipelines. 

There is also a problem with detection latency. If a dependency has already
shipped to production, simple deploy checking will not prevent remote code
execution if dependencies are automatically updated. Imagine the following timeline:

1. Node's dependency, `lpad` which pads text from the left is compromised. Evil actor releases a new version.
2. Your build releases a new version because it wants to update the copy on a specific case.
3. The version is pushed to prod.
4. The magic centralised database you're paying heaps for gets updated with the malicious dependency, but it's too late. 
5. Deployments start failing because of this transitive dependency.

So, SLSA needs to be accompanied by really rigorous restrictions in order to be
effective. For example, you may [pin your dependencies to a specific
version](https://maxleiter.com/blog/pin-dependencies), and that specific
version to a hash. This may be done automatically depending on your package
manager. You may decide to fork all repositories you depend on, and manually
build them from source. These kinds of things can be effective in mitigating
the risk of supply chain compromise, and SLSA can aid in verifying these things.

Additionally, there are other concerns with Sigstore's public infrastructure
that may be of concern. For example, the metadata associated with your binaries
is publicly exposed, which allows an attacker to gain valuable information
about what binaries you are producing and why. Hackers abusing certificate
transparency is [not a new phenomenon](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEF%20CON%2025%20-%20Hanno-Boeck-Abusing-Certificate-Transparency-Logs.pdf).

Additionally, Sigstore itself
may be compromised, which would allow an attacker to forge arbitrary
signatures. This is clarified in the [Threat Model - Sigstore](https://docs.sigstore.dev/about/threat-model/) page:

> Fulcio CA server
> 
> Arbitrary software attack: Can issue certs for any OIDC issuer/identity and use those to sign any software desired. Bob will see these certificates as valid as they are signed by the Fulcio CA and included in the Fulcio CT log.

So that's SLSA in 2000 words or fewer, from a technical perspective.
