---
coding: utf-8

title: Sideloaded HTTP Exchanges
docname: draft-yasskin-httpbis-sideloaded-exchanges-latest
category: std

ipr: trust200902

stand_alone: yes
pi: [comments, sortrefs, strict, symrefs, toc]

author:
 -
    name: Jeffrey Yasskin
    org: Google Inc.
    email: jyasskin@chromium.org

normative:
  OriginPolicy:
    target: https://wicg.github.io/origin-policy/
    title: Origin Policy
    author:
      - name: Mike West
        org: Google Inc.
    seriesinfo:
      Web Incubation Community Group: Draft Community Group Report
    date: 2017-05-30

informative:
  CSP: W3C.REC-CSP2-20161215
  SRI: W3C.REC-SRI-20160623
  SRICaching:
    target: https://hillbrad.github.io/sri-addressable-caching/sri-addressable-caching.html
    title: Subresource Integrity Addressable Caching
    author:
      name: Brad Hill
      organization: Facebook Inc.
    date: 2016-10-15

--- abstract

This document specifies how a server can provide an HTTP request/response pair
(an exchange) for another origin. The client verifies the exchange using a
public key fetched from the claimed origin.

--- note_Note_to_Readers

Discussion of this draft takes place on the HTTP working group mailing list
(ietf-http-wg@w3.org), which is archived at
<https://lists.w3.org/Archives/Public/ietf-http-wg/>.

The source code and issues list for this draft can be found in
<https://github.com/jyasskin/sideloaded-exchanges>.

--- middle

# Introduction {#intro}

Some clients have expensive data connections and would prefer to transfer the
data needed for a web application from a nearby peer instead of via the public
internet. These clients can usually afford to transfer small amounts of data,
which this specification uses to validate the data received from the peer.
However, those clients often have mobile data manually turned off, which will
prevent even this small amount of validation data from being transferred.

This draft refers to several components from
{{?I-D.yasskin-http-origin-signed-responses}}, but if this draft is accepted and
that one isn't, those components can be moved here.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.

# Side-loading an exchange {#side-loading}

An HTTP exchange can be stored in an `application/http-exchange+cbor` resource,
as defined in {{!I-D.yasskin-http-origin-signed-responses}}. When a client loads
this resource, it SHOULD record a claim that the origin server replies to the
wrapped request with the wrapped response. It MUST NOT store this pairing in the
normal HTTP cache ({{!RFC7234}}) without further validating it
({{exchange-validation}}).

The client MUST NOT validate a side-loaded exchange until its person indicates
some intention to visit the contained URL. See {{privacy}}.

# Validating a claimed exchange {#exchange-validation}

When a client attempts to navigate to a URL for which it has a side-loaded
exchange ({{side-loading}}), it MUST run the following steps to determine
whether to use the side-loaded response. If the steps return an exchange, the
client MUST use that exchange without fetching the URL itself. If they return
`null`, the client MUST fetch the URL over the network.

1. Let `publicKey` be the origin's signing key, determined by {{signing-key}}.
   If the client can't determine a signing key for the origin, return `null`.
1. Parse the side-loaded response's `Signature` header field into a list as
   described by {{!I-D.yasskin-http-origin-signed-responses}}, and validate the
   subset with `ed25519Key` properties matching `publicKey`, with the
   `allResponseHeaders` flag set.
1. If the validation procedure for any of these list elements returns
   "potentially-valid" with an `exchange` and `publicKey`, return `exchange`.
   Otherwise, return `null`.

# An origin's signing key {#signing-key}

The [origin policy
manifest](https://wicg.github.io/origin-policy/#origin-policy-manifest) concept
defined by {{OriginPolicy}} is extended with a "signingKey" member whose value
is a JSON object matching the following CDDL (Appendix E.1 of
{{!I-D.ietf-cbor-cddl}}):

~~~cddl
signingKey = {
  ed25519: tstr,
  expires: tstr,
  * tstr => any
}
~~~

`ed25519` is interpreted as a base64 string with padding omitted (Section 4 of
{{!RFC4648}}), encoding a 32-byte Ed25519 public key. If it's not present, has
characters not from the base64 alphabet, or is not 43 characters long, it is
invalid.

`expires` is interpreted as an {{!RFC3339}} date-time in which the "T" MUST be
uppercase, the time-secfrac MUST NOT be present, and the time-offset MUST be an
uppercase "Z". If these requirements aren't satisfied, the member MUST be
treated as not present. If the `expires` member is not present, in the past, or
more than 604800 seconds (7 days) in the future, it is invalid.

To determine an origin's signing key, the client MUST run the following steps:

1. Let `keyInfo` be `null`.
1. If the ["retrieve an Origin Policy for an
   origin"](https://wicg.github.io/origin-policy/#abstract-opdef-retrieve-an-origin-policy-for-an-origin)
   operation defined in {{OriginPolicy}} returns an origin policy object, and
   the origin policy object's body has a "signingKey" member, set `keyInfo` to
   the value of that member.
1. If the above origin policy was transferred over a connection guarded by a TLS
   certificate, and that certificate has expired or been revoked, set `keyInfo`
   back to `null`.
1. If `keyInfo` isn't `null`, but its `ed25519` member or `expires` member is
   invalid (see above), set `keyInfo` back to `null`.
1. If `keyInfo` is `null`, fetch the `/.well-known/origin-policy` path within
   the origin, following redirects, parse it as an origin policy manifest, and
   if  that manifest's "signingKey" member is present, set `keyInfo` to that
   member's value. The client MAY store an appropriate tuple in its Origin
   Policy Store.
1. If `keyInfo` is `null` or its `ed25519` or `expires` member is invalid (see
   above), the origin has no signing key.
1. Otherwise, the signing key is the base64-decoded bytes from
   `keyInfo.ed25519`.

# Security Considerations

## Confidential data

Authors MUST NOT include confidential information in a signed exchange that an untrusted intermediate could side-load, since the exchange is only signed and not encrypted. Intermediates can read the content.

## Downgrades

Signing a bad exchange can affect more users than simply serving a bad response,
since a served response will only affect users who make a request while the bad
version is live, while an attacker can side-load a signed exchange until its
signature or key expires. Authors SHOULD consider shorter signature expiration
times than they use for cache expiration times.

The author of an origin MAY rotate its signing key when they replace a
vulnerable resource. However, this isn't guaranteed to take effect until the old
signing key expires ({{signing-key}}).

Clients MAY also re-fetch an origin's Origin Policy more often than the signing
key expires, to check for such key rotation more aggressively.

## Unsigned headers

The `Signature` header ({{?I-D.yasskin-http-origin-signed-responses}}) doesn't
currently allow signing request headers (aside from the effective request URI,
which is required to be signed). This means that content negotiation isn't
trustworthy, even if multiple signed exchanges are available for the same URL.

It is also important to avoid using response headers that weren't signed, to
avoid, for example, type confusion attacks. The algorithm in
{{exchange-validation}} requires that clients use the exchange it returns,
rather than the original exchange, to avoid that risk.

## Invalid TLS certificates

The signing key is only trustworthy because it's transferred over a TLS
connection. If the certificate that guarded its transfer expires or is revoked,
trust in the signing key MUST be revoked as well. This is specified in
{{signing-key}} but may be difficult to implement.

## Stolen private keys

If the private key for the signing key is stolen, the thief can forge exchanges
for side-loading. Protect your keys.

# Privacy Considerations {#privacy}

Validating a side-loaded exchange ({{exchange-validation}}) reveals to that
exchange's original origin that the client has gone somewhere that gave it the
exchange, and that the client may be interested in actually reading the
exchange. The `Referer` header (Section 5.5.2 of {{?RFC7231}}) may reveal which
server provided the exchange. Validating a side-loaded exchange also reveals to
any network eavesdroppers (via SNI, Section 3 of {{?RFC6066}})) that the client
is interested in the origin server. This is the same information the origin and
network eavesdroppers would have received if the exchange were not side-loaded
and the client followed a link to it.

# IANA Considerations {#iana}

This document has no actions for IANA.

--- back

# Why didn't you ...

## Trust integrity attributes

{{CSP}}'s threat model is that an attacker may be able to inject a `<script
src='https://untrusted.example.com/script.js' integrity='ed25519-publickey'>`
element, but if the `Content-Security-Policy` header field doesn't include a
`script-src https://untrusted.example.com/` directive, that script won't
execute.

As discussed in {{SRICaching}}, allowing that attacker to claim that
`https://trusted.example.com/script.js` has particular content signed by a
particular public key, where `trusted.example.com` is chosen to already be
present in the resource's `script-src` directive, would allow them to run
arbitrary content, defeating the purpose of the `Content-Security-Policy`
header.

# Related work

This proposal is Eric Rescorla's counter-proposal to
{{?I-D.yasskin-http-origin-signed-responses}}. That proposal allows trust in
content from one origin but provided by another, without making a TLS connection
to the original origin. This proposal attempts to serve most of the same use
cases but preserves the requirement that the client, at some time in the past,
made a TLS connection to the original origin.

This proposal is related to proposals to cache resources with a key consisting
of their {{SRI}} hash.
<https://hillbrad.github.io/sri-addressable-caching/sri-addressable-caching.html>
discusses security risks of that caching proposal.

This proposal is also related to a proposal to side-load response payloads and
validate that content using a conditional HTTP request ({{?RFC7232}}). That
proposal isn't written yet, but it's sketched in
<https://docs.google.com/document/d/e/2PACX-1vSEzJ8PvAkx05YkfOuXoFjC6v8ByQ9Z11SNlP5pMGV8KXufrxkdhrbGmCzfr_NDsS-g-ISCjWOnkQBf/pub>.
