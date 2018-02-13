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
    organization: Google
    email: jyasskin@chromium.org

normative:
  OriginPolicy:
    target: https://wicg.github.io/origin-policy/
    title: Origin Policy
    author:
      org: WICG
    date: Living Standard

informative:
  SRI: W3C.REC-SRI-20160623

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

# Fetching a trusted public key

Read it from the origin's {{OriginPolicy}}.

# Security Considerations

--- back
