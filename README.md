# kotoba-lang/org-w3-json-ld-api

**[JSON-LD 1.1 expansion](https://www.w3.org/TR/json-ld11-api/), portable `.cljc`.**

Third layer of the `-rdfc-` cryptosuite stack. `org-w3-rdf-canon` canonicalizes an RDF
dataset; a Verifiable Credential is a JSON-LD document, and expansion is what turns its
context-dependent shorthand into the explicit form RDF conversion needs. Without it,
`eddsa-rdfc-2022` has nothing to canonicalize.

Complements `kotoba-lang/json-ld`, which provides EDN-first constructors and says in its
own README that the processor layer belongs elsewhere. This is that layer.

## Contexts are never fetched, and that is a security property

`expand` does no network I/O. A remote `@context` must be supplied in `:contexts`; an
unsupplied one throws `context-not-provided`.

A `@context` decides what every term in a document *means*. A verifier that fetches one
at verification time hands the context's host the power to change what a signature
covers — **after signing, with no key and no detectable tampering**. The document would
still verify, against a different graph. Pinning is the only way the meaning of a signed
document stays fixed, so this library cannot be configured to fetch.

```clojure
(require '[json-ld-api.core :as jld])

(jld/expand credential {:contexts {"https://www.w3.org/ns/credentials/v2" v2-context}})
```

## `expand` is NOT a validator

**30 of the suite's 103 malformed documents are still accepted rather than refused.** A
successful `expand` does not mean the input was valid JSON-LD. Validate separately.

## What refuses rather than guessing

Refused with `unsupported`: `@nest`, `@json` coercion, `@import`, `@propagate`,
`@included`, `@direction`, and the `@container` variants `@index`/`@id`/`@type`.

A processor that quietly ignored `@nest` would return an expansion **missing triples**,
and the caller's next step is to *sign* it — so the failure would surface as a signature
over a graph nobody intended, not as an error.

**Refusals fire at use, not at definition.** A large shared context defines terms a
document never touches; credentials/v2 declares `@json` on `_sd`, `cnf/jwk` and
`jsonSchema`. Refusing when those are merely *defined* made the whole context unusable.

## Measured, and the pass rate is not the useful number

```
official JSON-LD 1.1 expand suite (376 of 385 entries; 9 are 1.0-only)
  positive: 98/273 exact match   (97 refused as unsupported, 57 mismatch, 0 crash)
  negative: 45/103 exact error code (30 accepted that should have been refused)
```

The negative cases assert the spec's own error **code**, not merely that something was
raised. Counts are pinned as a floor, so they can only improve — but a baseline is not
approval, and the gaps print on every run.

What told me this library would refuse **every real Verifiable Credential** was expanding
one, not the pass rate: credentials/v2 puts `@container: @graph` on `proof`, and my
refusals sat at context-processing time. `a-real-verifiable-credential-expands` is the
test that matters, and it checks the datatypes a signature is actually taken over —
`cryptosuiteString`, `multibase`, `xsd:dateTime`, and `proofPurpose` as an IRI.

## Test

```bash
clojure -M:test     # includes the whole official suite, no network
clojure -M:lint
```

## License

MIT. See `LICENSE`.
