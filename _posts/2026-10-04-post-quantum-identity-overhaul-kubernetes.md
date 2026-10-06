---
layout: post
title: "the post-quantum identity overhaul in kubernetes"
date: 2026-10-04
image: /assets/images/post-quantum-identity.png
tags: [security, kubernetes, cryptography, post-quantum]
---

<figure class="lead uncropped">
  <img src="/assets/images/quantum-computer-illustrated.png" alt="two-buttons meme: a sweating figure agonising over which button to press">
  <figcaption><a href="https://www.reddit.com/r/ProgrammerHumor/comments/hbwrpw/quantum_computer_illustrated/">quantum computer, illustrated</a></figcaption>
</figure>

I want to talk about one of the hottest topics in the Kubernetes world of the last few years. Surprisingly, it has nothing to do with AI. It's the quiet, enormous undertaking of making a cluster's cryptography quantum-proof.

[Post-quantum cryptography](https://en.wikipedia.org/wiki/Post-quantum_cryptography) is by no means a recent innovation. Ciphers that shrug off [Shor's algorithm](https://en.wikipedia.org/wiki/Shor%27s_algorithm) have been around for decades — [Classic McEliece](https://classic.mceliece.org/) dates to 1978. So why are we still running the quantum-breakable classics — RSA, ECDSA, Diffie-Hellman? Mostly efficiency: McEliece is quantum-safe but its public keys run to about a megabyte, and as long as the quantum threat stayed theoretical, that trade was not worth making. Is it worth it now? Or rather, can we afford it?

That calculus has flipped. Quantum hardware keeps improving, and while a machine that can break RSA at real key sizes is probably still [some years out](https://postquantum.com/post-quantum/4099-qubits-rsa/), the migration itself takes years — so the clock is already running. NIST made it official: its 2024 draft [transition timeline](https://csrc.nist.gov/pubs/ir/8547/ipd) (IR 8547) proposes [deprecating](https://quantumsecuritydefence.com/insights/nist-algorithm-deprecation-timeline-rsa-ecc-sha1/) RSA, ECDSA and their peers around 2030 and disallowing them after 2035.

Back to Kubernetes: what actually has to change? Cryptography does two jobs: it keeps data confidential (encryption) and it proves identity (signing). The confidentiality half is already handled: the hybrid [post-quantum key exchange](https://en.wikipedia.org/wiki/Hybrid_cryptosystem) `X25519MLKEM768` has been the default for control-plane TLS since Kubernetes 1.33 (April 2025) and is covered well by the [July 2025 Kubernetes PQC blog](https://kubernetes.io/blog/2025/07/18/pqc-in-k8s/); the cloud providers are shipping the same hybrid on their own service endpoints too ([AWS's KMS and Secrets Manager among them](https://aws.amazon.com/blogs/security/protecting-your-secrets-from-tomorrows-quantum-risks/)). The identity half — the signatures that decide whether a token or certificate is genuine, and whose failure lets an attacker *forge* them — is the one still unfinished, and still undocumented. There is a good reason it lags behind. But to get to it, we need a quick stop in the theory corner.

## just enough cryptography

There is already a wealth of excellent material on all of this (papers, courses, conference talks), so I will not re-derive the fundamentals here; this is the *just enough* version to follow the Kubernetes story, and where a later section leans on a specific idea I pull it back in then. I also link generously as we go, and the material behind those links is well worth a deep dive on its own.

### why quantum breaks the classical layer (and not all of it)

Classical public-key cryptography rests on two problems that are easy one way and believed hard the other: factoring a large integer, and the discrete logarithm. RSA leans on the first; Diffie-Hellman, ECDH and ECDSA lean on the second. Shor's algorithm solves **both** efficiently on a quantum computer. That single result is what puts the entire classical public-key layer — encryption, key exchange *and* signatures — on notice.

Symmetric cryptography is in far better shape. The best known quantum attack on a block cipher like AES is [Grover's algorithm](https://en.wikipedia.org/wiki/Grover%27s_algorithm), which gives only a quadratic speedup — it effectively halves the key length. The answer is boring: use a longer key. AES-256 stays comfortably safe; SHA-256/SHA-3 stay safe. So the migration is not "replace all cryptography". It is specifically **the public-key layer**: the key agreement and the signatures. Everything symmetric comes along unchanged.

That split is why the problem has two independent fronts:

- **KEMs** (key-encapsulation mechanisms) replace RSA-encryption, DH and ECDH: the confidentiality / key-exchange job.
- **Signatures** replace RSA-sign and ECDSA: the authenticity / identity job.

### where the new schemes come from: lattices

[NIST](https://csrc.nist.gov/projects/post-quantum-cryptography) ran a multi-year competition and standardised the first post-quantum schemes in August 2024; see [this talk](https://www.youtube.com/watch?v=aw6J1JV_5Ec) for the story of that 8-year-long battle-royale:

- [**FIPS 203 — ML-KEM**](https://csrc.nist.gov/pubs/fips/203/final) (formerly [*CRYSTALS-Kyber*](https://pq-crystals.org/kyber/)): the key-encapsulation mechanism.
- [**FIPS 204 — ML-DSA**](https://csrc.nist.gov/pubs/fips/204/final) (formerly [*CRYSTALS-Dilithium*](https://pq-crystals.org/dilithium/)): the signature scheme. **This is the one this post is about.**
- [**FIPS 205 — SLH-DSA**](https://csrc.nist.gov/pubs/fips/205/final) (*SPHINCS+*): a conservative hash-based signature backup.

ML-KEM and ML-DSA are both **[lattice-based](https://blog.cloudflare.com/lattice-crypto-primer/)**. A lattice is a regular grid of points spanning a high-dimensional space, and their security rests on [lattice problems](https://en.wikipedia.org/wiki/Lattice_problem) such as the [shortest-vector problem](https://www.youtube.com/watch?v=QDdOoYdb748) (SVP) and the closest-vector problem (CVP): find the shortest non-zero point in the grid, or the grid point nearest some arbitrary target. Those are believed hard even for quantum computers. Shor does not help because, unlike factoring, they expose no hidden periodic structure for it to exploit, so a quantum computer gets only a generic, polynomial speedup. In practice the schemes stand on the [learning-with-errors](https://en.wikipedia.org/wiki/Learning_with_errors) family (Module-LWE for Kyber, Module-SIS for Dilithium), a structured variant that keeps keys and operations fast.

It is worth picturing a lattice key pair concretely, because it comes back later: the **public key** is a chunk of pseudo-random-looking bytes derived from the lattice (1952 of them for ML-DSA-65), and the **private key** is generated from, and can be stored as, a 32-byte seed. There is no modulus and no curve, just a byte string, which is exactly why, further down, [JOSE](https://www.rfc-editor.org/info/rfc7517) needs a new way to carry it.

And there is a price, the recurring theme of the practical half of this post: **the objects are big.** That 1952-byte public key and a 3309-byte signature sit against 32 and 64 bytes for Ed25519. The size shows up in every token, every certificate and every handshake.

### the go angle

This whole story is buildable today because the Go standard library grew the primitives in lock-step:

- **Go 1.24** (February 2025) added `X25519MLKEM768` to [`crypto/tls`](https://pkg.go.dev/crypto/tls), the KEM half that Kubernetes 1.33 now defaults to.
- **Go 1.27** (August 2026) added [`crypto/mldsa`](https://pkg.go.dev/crypto/mldsa), and [`crypto/x509`](https://pkg.go.dev/crypto/x509) learned to marshal and parse ML-DSA keys, certificates and CSRs: the signature half.

That second line matters more than it looks, and it is worth being explicit about what I mean by it. Before native support, using ML-DSA from Go meant `cgo` bindings to a C library like [liboqs / Open Quantum Safe](https://openquantumsafe.org/) — a build-time and supply-chain dependency that most projects will not take on for a core security path. With Go 1.27 the algorithm is in the standard library, so Kubernetes, go-jose and everything downstream can adopt it as plain Go with nothing extra to vendor. Every piece of working code in this post is pure stdlib Go — no C, no external crypto library.

## post-quantum cryptography in kubernetes

### where cryptography lives in a cluster

Split the cluster's cryptography the same two ways, encryption and signing, and the migration picture falls out.

**Encryption / key exchange** is every TLS connection between components: the API server to etcd, the kubelet to the API server, every `client-go` client. This is the confidentiality half, and it is the part Kubernetes has already moved — `X25519MLKEM768` hybrid key exchange is the default since 1.33. I will not spend more time on it.

**Signing / identity** is the frontier, and it is where a cluster's trust actually lives:

- **ServiceAccount tokens**: short-lived JWTs the API server signs and projects into every pod. A workload uses one to call the API, and (through OIDC federation) to prove who it is to systems *outside* the cluster, like AWS IRSA or Vault. These are the subject of the rest of the post.
- **OIDC verification**: the other end of those tokens: anything that fetches the cluster's published keys and checks a token's signature itself, from a cloud IAM to a policy engine.
- **the cluster PKI and image signing**, the long-lived anchors: the CA that signs component and client certificates, and the keys that sign container images.

From here on it is one question: the ServiceAccount token, and who verifies it.

### state of the art: a very active frontier

Look at the dates before anything else. The signature primitive became a NIST standard in **August 2024**, landed in the Go standard library with **Go 1.27 in August 2026**, got its JOSE binding (**RFC 9964**) in May 2026, and the Kubernetes token work is *still unmerged* as I write this in **late 2026**. This is wet concrete.

The signature work is tracked under one umbrella issue, [kubernetes #141838](https://github.com/kubernetes/kubernetes/issues/141838), and it is a dependency chain more than a single feature:

```
Go 1.27 crypto/mldsa
   └─ go-jose v4 gains ML-DSA  (issue #272, PR #282)
        └─ go-oidc / client-go
             └─ apiserver OIDC authenticator  +  SA-token signer (PR #142397)
```

Two things are landing in **Kubernetes 1.38-alpha** already:

- **certificates / CSRs.** ML-DSA PodCertificateRequests and CSRs, behind the feature gate `CertificateSigningRequestMLDSA`, with client-go and kubelet able to use ML-DSA keys. The whole X.509 path works on the Go 1.27 stdlib.
- **the OIDC authenticator** learning to *verify* ML-DSA-signed tokens.

The piece that is **not** merged, and the most interesting one, is the ServiceAccount-token **signer**: [kubernetes PR #142397](https://github.com/kubernetes/kubernetes/pull/142397), "ML-DSA support for tokens". It teaches the apiserver to *sign* SA tokens with ML-DSA, adds the `ServiceAccountTokenMLDSA` feature gate, and publishes ML-DSA public keys in the cluster's JWKS. It is a draft, because it depends on something upstream of Kubernetes entirely.

That dependency is the real bottleneck: [**go-jose #282**](https://github.com/go-jose/go-jose/pull/282) (tracked by [#272](https://github.com/go-jose/go-jose/issues/272)), opened September 2026 and still open. Kubernetes signs and verifies JWTs through `go-jose`, and the version it vendors (v4.1.5) has no ML-DSA. PR #282 adds it, built on the Go 1.27 `crypto/mldsa` stdlib. Until it merges and Kubernetes bumps the dependency, #142397 stays a draft. One unmerged library PR gates the whole SA-token story, and, as we will see, a good chunk of the ecosystem beyond Kubernetes too.

### rfc 9964: JWT token evolution

A ServiceAccount token is a JWT in compact JWS form: three base64url segments, `header.payload.signature`:

```
eyJhbGciOiJSUzI1NiIsImtpZC… . eyJpc3MiOiJodHRwczovL2t1Ym… . <signature>
└──────── header ─────────┘   └──────── payload ────────┘   └── sig ──┘
```

Going post-quantum changes what the token carries: the algorithm and the signature. The payload, the claims that say *who* the token is for, does not change.

Before any of this could be expressed, JOSE had a gap. A JWK (the JSON object in a JWKS) encodes a key using fields specific to its maths: RSA keys carry a modulus and exponent (`n`, `e`), elliptic-curve keys carry a curve name and a point (`crv`, `x`, `y`), Ed25519 keys (`OKP`) carry `crv` and `x`. An ML-DSA key has none of those shapes: no modulus, no curve, no point, just an opaque FIPS-204 byte string. It did not fit any existing JWK type.

[RFC 9964](https://www.rfc-editor.org/info/rfc9964/) ("ML-DSA for JOSE and COSE", published May 2026) fixes that with a deliberately generic key type, **`AKP`** (*Algorithm Key Pair*). An AKP key carries the raw public key in a single `pub` member (base64url), the 32-byte seed in `priv` for a private key, and leans on `alg` to say *which* algorithm, since there is no `crv` to disambiguate the parameter set. It also registers three JWS `alg` values: **`ML-DSA-44`**, **`ML-DSA-65`**, **`ML-DSA-87`**.

So the concrete before/after. Here is a classical ServiceAccount token **today**: its JWS header, and the JWK the issuer publishes so others can verify it:

```js
// JWS header
{ "alg": "RS256", "kid": "sa-key-1" }

// matching JWKS entry
{ "kty": "RSA", "alg": "RS256", "use": "sig", "kid": "sa-key-1",
  "n": "0vx7agoebGcQSuu…",   // 256-byte RSA modulus
  "e": "AQAB" }   // public exponent (65537)
```

The **post-quantum** version. Only the algorithm, the key type and the key material move; the `kid` and everything else stay put:

```js
// JWS header
{ "alg": "ML-DSA-65", "kid": "sa-key-1", "typ": "JWT" }

// matching JWKS entry (AKP, RFC 9964)
{ "kty": "AKP", "alg": "ML-DSA-65", "use": "sig", "kid": "sa-key-1",
  "pub": "olQb1NoihSGc3sfQ…" }   // the 1952-byte ML-DSA public key
```

That `pub` member is exactly the lattice public key from the theory section: the raw ML-DSA key bytes, base64url-encoded. Because an ML-DSA key is just a byte string, `AKP` has nothing like RSA's `n`/`e` or an elliptic curve's `x`/`y`; the whole key is the one `pub` field, and `alg` says which parameter set it belongs to. The third JWS segment, the signature, swaps 256 bytes of RSA for 3309 bytes of ML-DSA. The payload — the claims that say who the token is for (`sub`, `aud`, `exp`, the `kubernetes.io` block) — is byte-for-byte identical.

That is the whole shape of the change, and why it is deceptively large. The token's *meaning* is identical; what moves is the `alg`, the signature bytes, and the key type in the JWKS. But every consumer that verifies a token now has to do three new things: recognise `ML-DSA-65` as a signing algorithm, parse a `kty: AKP` JWK into an ML-DSA public key, and call an ML-DSA verify routine instead of RSA. A verifier that knows only `RS256`/`ES256` does not degrade gracefully: it sees an `alg` it has never heard of and rejects the token outright. Multiply that across every system that checks a cluster token and you have the migration.

## building it for real

The theory is easy to state and easy to doubt, so I built it: a real Kubernetes API server signing real ServiceAccount tokens with ML-DSA, ahead of the feature merging.

### making a cluster sign tokens with ML-DSA

You cannot wait for #142397 to merge to try it, so you build Kubernetes straight from the pull request. The maintainers' own workflow compiles the tree into a [kind](https://kind.sigs.k8s.io/) node image (`kind build node-image --type source`) and boots a cluster from it; point it at the PR branch and, about half an hour of compiling later, you have a `v1.38.0-alpha` node image with the ML-DSA token signer in it.

Then comes the part that surprised me, and it is the finding worth keeping:

> Enabling the `ServiceAccountTokenMLDSA` feature gate does nothing on its own.

The API server signs ServiceAccount tokens with whatever key file it is handed (`--service-account-signing-key-file`). The signing algorithm is chosen purely from the *type* of that key: an RSA key yields RS256, an ML-DSA key yields ML-DSA. The gate is only a guard: in the signer's type switch, the ML-DSA branch refuses to run unless the gate is on. But kubeadm — which `kind` uses — generates an **RSA** key pair by default. So a stock cluster with the gate flipped still emits RS256. Gate on + RSA key = RS256. Gate on + ML-DSA key = ML-DSA. **Only the key changes the algorithm; the gate only unlocks it.**

The fix is to hand the API server an ML-DSA signing key. Go 1.27's `crypto/x509` marshals an ML-DSA key to standard PKCS#8, and Kubernetes' `keyutil` parses it back with no code change at all, which is why PR #142397 touches no key-loading code. So: generate an ML-DSA `sa.key` / `sa.pub`, and because kubeadm only generates a key pair when one is *absent*, bind-mount them into `/etc/kubernetes/pki/` before init so kubeadm adopts them. Gate on, ML-DSA key in place, and the cluster signs with ML-DSA.

The result, from a real `kubectl create token`:

```
$ kubectl create token default | cut -d. -f1 | base64 -d
{"alg":"ML-DSA-65","kid":"AGYWxRt2vJoGnwfl4YeWya93…","typ":"JWT"}

$ kubectl get --raw /openid/v1/jwks
{"keys":[{"use":"sig","kty":"AKP","alg":"ML-DSA-65","kid":"AGYWxRt2…","pub":"olQb1…"}]}

$ kubectl get --raw /.well-known/openid-configuration | jq .id_token_signing_alg_values_supported
["ML-DSA-65"]
```

A real Kubernetes API server, signing ServiceAccount tokens with a post-quantum signature, publishing the public half as an `AKP` JWKS. That is the whole feature, working, ahead of its own merge.

### performance: bytes, not CPU, and a lost HTTP/2 feature

The first surprise is that CPU is a non-issue. ML-DSA-65 signs in ~266 µs (faster, as it happens, than RSA-2048) and verifies in ~72 µs, and the API server caches token authentications anyway, so signatures are not re-checked on every request. The whole cost is **size**. An ML-DSA-65 ServiceAccount token is about **4.9 KB against the ~800-byte RS256 token a cluster signs today, roughly 6×**. (The bound, projected token lives in the pod's tmpfs and in the request header, not in etcd, so that growth lands on the wire, not in storage.)

Most of that 6× you can reason about: a few KB more in each `Authorization` header, a bigger JWKS to publish. The cost that is not obvious is what the size *destroys*.

**HPACK, "[the silent killer feature of HTTP/2](https://blog.cloudflare.com/hpack-the-silent-killer-feature-of-http-2/)", is switched off without a word.** Kubernetes API traffic is HTTP/2, which compresses headers with HPACK: a per-connection dynamic table that, after the first request, sends a repeated header like `Authorization: Bearer …` as a ~1–2 byte table reference instead of the whole value. That table defaults to **4096 bytes**, and a field larger than the table can never be inserted ([RFC 7541 §4.4](https://www.rfc-editor.org/rfc/rfc7541#section-4.4)). The ~800-byte classical token fits, is indexed once, and is effectively free from then on. **A ~4.9 KB ML-DSA token does not fit, so it is re-sent in full on every single request**: a feature that is doing real work for you today simply stops, and there is no switch to turn it back on.

The loss is easy to measure directly. Mint a token on each cluster and count the HPACK-encoded header bytes a client writes over one reused connection, on a stable v1.37 API server for RS256 and the #142397 edge cluster for ML-DSA-65:

| token | size | 1st request | steady / request | 1000 requests |
|-------|-----:|------------:|-----------------:|--------------:|
| RS256 (v1.37) | 930 B | 821 B | **16 B** | ~17 KB |
| ML-DSA-65 (#142397) | 5005 B | 4155 B | **~4039 B** | ~4.0 MB |

Once the connection is warm, the classical token's whole header block collapses to **16 bytes**: it is in the table, sent as a short index. The ML-DSA token never gets in, so it ships in full on every request: about **4 KB each, ~250×** the classical cost, and **~17 KB versus ~4 MB** over a thousand calls (ML-DSA-87 is larger still). It is ~4 KB rather than the raw ~5 KB only because HPACK still Huffman-compresses the literal by about 20%; it just cannot *index* it away. Push the dynamic table above the token size (as the offline benchmark in the repo does, since the API server fixes it at 4096 and exposes no knob) and the token indexes again, dropping straight back to a handful of bytes: proof it is the 4096-byte ceiling, not any inherent incompressibility.

What turns that into real cost is how Kubernetes uses connections: they are **heavily reused**. client-go holds one long-lived, HTTP/2-multiplexed connection per client and runs essentially all of its traffic over it for the lifetime of the process, so a classical token is learned once and is near-free for every request after. ML-DSA erases that amortization and turns it into a steady per-request tax, an extra few KB on *every* call, landing on the single busiest path in the cluster, the API server, much of it crossing availability zones. Long-lived watches pay it only once per stream, so the cost concentrates on discrete GET/LIST/POST calls, and on the reconnect storm when the API server restarts and every client re-establishes at once.

Separately, the raw size also brushes against smaller limits at the edges: a ~4 KB cookie a 5 KB token cannot fit (anything cookie-based, like oauth2-proxy), or the 8–10 KB header caps on some ingresses, gateways and CDNs (nginx, AWS API Gateway, CloudFront). Those are mostly tunable defaults and the API server itself absorbs the token fine, so the load-bearing cost stays the HPACK loss on the reused connection above.

There is no clean operator-side fix. Raising the HPACK table past the token would restore indexing, but the API server fixes it at 4096 and exposes no knob, and a bigger table costs memory on every connection anyway. The real fix is a smaller signature — a scheme like [FN-DSA / Falcon](https://quantumsequrity.com/blog/fn-dsa-falcon-explained), whose ~666-byte signatures (about 3.6× smaller than ML-DSA's) would sit comfortably under the 4 KB table — but that is a long way off: [FIPS 206](https://csrc.nist.gov/News/2024/postquantum-cryptography-fips-approved) is **still unpublished** (expected around 2027), Falcon's floating-point sampling makes a safe constant-time implementation hard, Go has no plan to ship it ([golang/go#64537](https://github.com/golang/go/issues/64537) waits for a finalised standard, as it did for ML-DSA in Go 1.27), and it would still need a JOSE registration since [RFC 9964](https://www.rfc-editor.org/info/rfc9964/) covers only ML-DSA. So for the foreseeable future ML-DSA is the one option JOSE and Kubernetes implement — and this is the bill.

### what it means downstream: the real work is the verifiers

Here is the part that makes this an *overhaul* rather than a feature. Once ML-DSA is available in the JWT/JOSE stack, **signing** a post-quantum token is the easy side: any single issuer, holding the new key, is a few hundred lines. The migration proper is everything that **verifies** the token, and that is distributed across the whole cloud-native ecosystem.

The deciding question for any consumer is *how* it checks a token.

One kind **verifies the signature itself**, fetching the issuer's JWKS and doing the crypto, and this is the group that must change. Unpatched, it reads the header, sees `alg: ML-DSA-65`, and fails closed on an algorithm it does not know. The list is long: AWS STS (`AssumeRoleWithWebIdentity`, i.e. IRSA), GCP / Azure Workload Identity, HashiCorp Vault's JWT auth method, Dex, Keycloak, OPA / Gatekeeper, and Envoy's `jwt_authn` filter (the biggest gap — C++/BoringSSL, no tracker). Most of the Go ones route through `go-jose` and `coreos/go-oidc`, which is why go-jose #282 is the chokepoint for the whole ecosystem, not just Kubernetes. The work is already surfacing in the projects themselves; Keycloak, for one, now tracks it under a dedicated [post-quantum readiness umbrella](https://github.com/keycloak/keycloak/issues/43690).

The other kind **never checks the signature at all**, so it needs no change. Instead of doing the crypto, these hand the whole token to the API server's `TokenReview` API and take back a plain valid/invalid answer (SPIRE, Istio, Linkerd, Vault's *Kubernetes* auth method, Secrets Store CSI), or they simply forward the token untouched and let the API server authenticate it (Argo, Flux, Prometheus). Since the API server does the verifying — and #142397 already teaches it ML-DSA — this entire group keeps working with nothing to patch. That is the reassuring half of the blast radius, and it is most of the mesh and identity stack: worth stating so nobody wastes time trying to "upgrade" SPIRE or Istio for PQC.

The change in verifiers is small: recognise the new algorithm, turn the `AKP` JWK into an ML-DSA public key, and verify the signature. I wrote that missing verifier to show it; on the Go 1.27 standard library it comes down to this:

```go
// An AKP JWK pulled from the issuer's /openid/v1/jwks:
//   { "kty":"AKP", "alg":"ML-DSA-65", "kid":"sa-key-1", "pub":"<base64url>" }

// 1. decode "pub" into an ML-DSA public key — this is the new step a
//    classical verifier has no code path for
raw, _ := base64.RawURLEncoding.DecodeString(jwk.Pub)
pub, err := mldsa.NewPublicKey(mldsa.MLDSA65(), raw)

// 2. verify the JWS. signingInput is  base64url(header) + "." + base64url(payload);
//    sig is the decoded third segment.
err = mldsa.Verify(pub, []byte(signingInput), sig, &mldsa.Options{})
```

That is the entire capability a consumer is missing: a `kty` and an `alg` it does not recognise, and one `mldsa.Verify` call. [go-jose #282](https://github.com/go-jose/go-jose/pull/282) packages the same thing behind the normal JOSE API, so once it merges most Go verifiers inherit it from a dependency bump plus one allow-list entry.

The demo drives it home with a works-versus-fails contrast. A stock verifier, one whose algorithm allow-list has no ML-DSA entry, fed a real cluster token, rejects it outright:

```
unexpected signature algorithm "ML-DSA-65"; expected ["RS256" "ES256" …]
```

The patched verifier, the dozen lines above, accepts the same token. That rejection, repeated across every system in the first bucket, *is* the migration.

### a note on managed clusters

On EKS, GKE or AKS you control none of this: the control plane, the signing key and the feature gates are the provider's. The rollout will most likely run in this order: their own token verifier learns ML-DSA first (AWS STS, for IRSA), since they can't sign what their own services can't yet verify; then a **dual-key rotation**, with the JWKS carrying both the RSA and the ML-DSA key for an overlap window while new tokens sign ML-DSA and the short-lived old ones expire out on their own; and it ships **opt-in per cluster** well before it is ever a default — the same cautious arc `X25519MLKEM768` took.

## takeaway

Post-quantum **identity** in Kubernetes is underway. The key-exchange half already shipped and is the default; the signature half, ML-DSA for ServiceAccount tokens and certificates, is being built right now under [#141838](https://github.com/kubernetes/kubernetes/issues/141838) and [#142397](https://github.com/kubernetes/kubernetes/pull/142397), already working ahead of its own merge. The cost to plan around is **size, not CPU**: ML-DSA signs and verifies cheaply, but its ~4 KB signature overruns HPACK's 4096-byte header table and strips the compression off the busiest path in the cluster, a steady per-request tax on API-server traffic. And signing is only one side of it; the other, easy to forget, is **the verifiers**: every offline JWKS consumer, from the cloud IAMs (STS/IRSA, GCP/Azure WIF) to the CNCF projects (Keycloak, Dex, OPA, Envoy), has to learn ML-DSA too, and that work is already moving across the ecosystem behind the [go-jose #282](https://github.com/go-jose/go-jose/pull/282) chokepoint. None of it is optional or far-off: [NIST's transition timeline](https://csrc.nist.gov/pubs/ir/8547/ipd) deprecates the classical algorithms around 2030 and disallows them after 2035, so the whole migration runs against a real, **dated deadline**.

The benchmark, the reference verifier and the scripts that build a #142397 cluster are in [github.com/xvirgov/pqc-k8s](https://github.com/xvirgov/pqc-k8s).
