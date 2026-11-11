---
title: JSON Web Key (JWK) for DNSSEC
abbrev: JWK for DNSSEC
docname: draft-schlyter-dnssec-jwk-latest

stand_alone: true
v: 3

submissiontype: IETF
ipr: trust200902
kw: Internet-Draft
cat: std

pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
  - name: Jakob Schlyter
    org: Kirei AB
    email: jakob@kirei.se

normative:
  RFC1035:
  RFC4034:
  RFC7517:
  RFC7518:
  RFC7638:
  RFC8037:
  RFC8259:
  IANA-DNSSEC-ALG:
    target: https://www.iana.org/assignments/dns-sec-alg-numbers
    title: Domain Name System Security (DNSSEC) Algorithm Numbers
    author:
      org: IANA
  IANA-JOSE:
    target: https://www.iana.org/assignments/jose
    title: JSON Object Signing and Encryption (JOSE)
    author:
      org: IANA

informative:
  I-D.ietf-cose-dilithium:
  RFC4033:
  RFC7516:
  RFC8792:
  RFC9364:

--- abstract

This document specifies how to represent DNSSEC public and private keys as JSON Web Keys (JWK). The cryptographic key material is encoded using the existing JWK key types, and the DNS-specific information needed to reconstruct the corresponding DNSKEY resource record (owner name, algorithm, protocol and flags) is carried in a new JWK parameter, "dnskey". This gives DNSSEC signers, key management systems and other tools a standardized, interoperable format for exchanging and storing DNSSEC keys.

--- middle

# Introduction

DNS Security Extensions (DNSSEC) {{RFC9364}} use public key cryptography to sign DNS data. The public keys are published in the DNS as DNSKEY resource records {{RFC4034}}, but there is no standardized format for storing or exchanging the corresponding private keys. In practice, implementations use their own formats, the most widespread being the BIND "Private-key-format" text files, which have never been formally specified. This makes it unnecessarily hard to move keys between signers, key management systems and tooling, for example when changing DNS operator or signer implementation.

JSON Web Key (JWK) {{RFC7517}} is a well-established and widely implemented JSON {{RFC8259}} format for representing cryptographic keys, with defined encodings for RSA, elliptic curve, Edwards-curve and ML-DSA keys {{RFC7518}} {{RFC8037}} {{I-D.ietf-cose-dilithium}}, covering the signature schemes used by DNSSEC. A JWK alone, however, does not carry the information needed to construct a DNSKEY resource record: the owner name, DNSSEC algorithm number, flags and protocol fields.

This document defines a profile of JWK for DNSSEC keys. A DNSSEC key is represented as a plain JWK, with the key material encoded as specified for its JWK key type, and the DNSKEY metadata included in a new JWK parameter named "dnskey". Since the result is a valid JWK, existing JWK libraries can be used to parse and produce such keys, and applications that do not understand the "dnskey" parameter can still use the key material.

## Requirements Language

{::boilerplate bcp14-tagged}

## Terminology

This document uses DNSSEC terminology as defined in {{RFC4033}} and {{RFC4034}}, and JWK terminology as defined in {{RFC7517}}.

DNSSEC JWK:
: A JWK that represents a DNSSEC key and contains a "dnskey" parameter as specified in this document.

Producer:
: An entity that creates a DNSSEC JWK, for example a key generator or a tool exporting keys from a signer.

Consumer:
: An entity that parses a DNSSEC JWK, for example a signer importing a key.

# DNSSEC Keys as JSON Web Key

A DNSSEC JWK is a JWK as defined in {{RFC7517}}, consisting of:

1. the key material for the key, encoded using the JWK key type and parameters corresponding to the DNSSEC algorithm, as specified in {{algorithms}};
2. a "dnskey" parameter, as specified in {{dnskey-parameter}}; and
3. optionally, other JWK parameters, as specified in {{other-parameters}}.

A DNSSEC JWK representing a private key contains both the public and private key parameters for the key type. A DNSSEC JWK representing a public key contains only the public key parameters. A public DNSSEC JWK can be derived from a private DNSSEC JWK by removing the private key parameters, as described in {{Section 6 of RFC7518}} and {{RFC8037}}.

## Key Material {#algorithms}

DNSSEC algorithms are identified by the numbers registered in the "DNS Security Algorithm Numbers" registry {{IANA-DNSSEC-ALG}}, and the specification referenced for each algorithm in that registry defines its underlying signature scheme and DNSKEY public key encoding. The key material of a DNSSEC key MUST be encoded using the JWK key type ("kty") registered in the "JSON Web Key Types" registry {{IANA-JOSE}} for that signature scheme, together with any key type specific parameters, such as "crv" for elliptic curve keys. For example:

* RSA keys use the "RSA" key type ({{Section 6.3 of RFC7518}}).
* ECDSA keys use the "EC" key type, with "crv" identifying the curve ({{Section 6.2 of RFC7518}}).
* EdDSA keys use the "OKP" key type, with "crv" identifying the curve {{RFC8037}}.
* ML-DSA keys use the "AKP" key type, with "alg" identifying the parameter set {{I-D.ietf-cose-dilithium}}.

A DNSSEC algorithm for which no corresponding JWK key type is registered cannot be represented using this specification.

The JWK key types do not, in general, identify the DNSSEC algorithm; for example, the same RSA key could be used with algorithm 8 (RSASHA256) or 10 (RSASHA512). The DNSSEC algorithm is therefore always given by the "algorithm" member of the "dnskey" parameter.

For RSA private keys, Producers MUST include all of the private key parameters "d", "p", "q", "dp", "dq" and "qi" ({{Section 6.3.2 of RFC7518}}). Producers MUST NOT include the "oth" parameter.

The public key field of the DNSKEY RDATA is derived from the JWK public key parameters using the encoding specified for the DNSSEC algorithm.

## The "dnskey" Parameter {#dnskey-parameter}

The "dnskey" parameter is a JSON object holding the fields of the DNSKEY resource record that are not part of the key material, optionally along with the key tag. It contains the following members, all of which are REQUIRED unless stated otherwise:

name:
: The owner name of the DNSKEY resource record, as a JSON string. The name MUST be fully qualified (i.e., end with a "."), and MUST be in DNS presentation format as specified in {{Section 5.1 of RFC1035}}, with octets that are not printable ASCII, as well as ".", "\\", and other special characters within a label, represented using the "\\DDD" escapes. Note that, in JSON, the backslash itself is escaped, so the label "b\\000rken" appears as "b\\\\000rken" in the serialized JWK.

flags:
: The value of the Flags field ({{Section 2.1.1 of RFC4034}}), as a JSON integer in the range 0 to 65535.

protocol:
: The value of the Protocol field ({{Section 2.1.2 of RFC4034}}), as a JSON integer. This MUST be 3.

algorithm:
: The DNSSEC algorithm number ({{Section 2.1.3 of RFC4034}}), as a JSON integer.

keytag:
: OPTIONAL. The key tag of the DNSKEY resource record, as computed per {{Appendix B of RFC4034}}, as a JSON integer.

The key tag is redundant, since it can be computed from the other fields and the key material. Producers SHOULD include it to make the key easy to identify for humans and tools (e.g., to match a key with a DS record or RRSIG) without performing that computation, and to allow Consumers to detect inconsistencies.

Consumers MUST ignore unrecognized members of the "dnskey" object.

## Key Timing and Other Metadata

Key management metadata, such as the times at which a key was created or is to be published, activated, retired or removed, is out of scope for this specification. Such information is not part of the DNSKEY resource record and is therefore not included in the "dnskey" parameter. It could be specified in the future as a separate JWK parameter.

## Other JWK Parameters {#other-parameters}

Producers SHOULD include a "kid" (Key ID) parameter ({{Section 4.5 of RFC7517}}). When included, it is RECOMMENDED that its value be the JWK Thumbprint {{RFC7638}} of the key, computed using SHA-256. Note that the "kid" is not the same as the DNSSEC key tag,
which is not unique and is optionally carried in the "dnskey" parameter.

The "alg" parameter ({{Section 4.4 of RFC7517}}) is OPTIONAL, except where required by the key type. When included, it MUST be consistent with the DNSSEC algorithm.

Other JWK parameters, such as "use" or "key_ops", MAY be included. Consumers MAY ignore them.

## Processing Rules

A Consumer importing a DNSSEC JWK MUST:

1. verify that the "dnskey" parameter is present and contains all required members with values of the correct type;
2. verify that the JWK key type and its parameters (e.g., "crv" or "alg") are consistent with the DNSSEC algorithm;
3. construct the DNSKEY RDATA from the "flags", "protocol" and "algorithm" members and the public key material;
4. if the "keytag" member is present, compute the key tag of the resulting DNSKEY RDATA and verify that it is equal to the "keytag" member (if specified); and
5. for a private key, verify that the private key corresponds to the public key.

If any of these checks fail, the Consumer MUST reject the key.

## Key Sets

Multiple DNSSEC JWKs, for example all the keys of a zone, MAY be represented as a JWK Set ({{Section 5 of RFC7517}}), where each key in the set carries its own "dnskey" parameter.

# Examples

This section shows a public DNSSEC JWK. Example private DNSSEC JWKs for a selection of DNSSEC algorithms, together with their DNSKEY resource records, are given in {{example-keys}}.

The following public key is derived from the ECDSA Curve P-256 with SHA-256 private key in {{example-ecdsap256sha256}} by removing the private key parameter "d":

~~~ json
{
  "kid": "ni1azkx1Qe3zOuUw1eRd3oNN_91av68ll49pGTLF_BM",
  "kty": "EC",
  "crv": "P-256",
  "x": "9cqheie08FXlvnU17YJJm1FMUXD4rwi7-4atSdZ3og8",
  "y": "PR66xkQK4YKumafBX4tYV4N008RfZHQZdMxs_7ToK5A",
  "dnskey": {
    "name": "example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 13,
    "keytag": 9115
  }
}
~~~
{: title="ECDSA Curve P-256 with SHA-256 Public Key"}

# IANA Considerations

IANA is requested to register the following parameter in the "JSON Web
Key Parameters" registry {{RFC7517}}:

Parameter Name:
: dnskey

Parameter Description:
: DNSSEC DNSKEY Parameters

Used with "kty" Value(s):
: *

Parameter Information Class:
: Public

Change Controller:
: IETF

Specification Document(s):
: {{dnskey-parameter}} of this document

# Security Considerations

A DNSSEC JWK containing a private key is sensitive, and compromise of it allows an attacker to sign arbitrary data for the zone. Such keys MUST be protected against unauthorized access, both when stored and when transmitted. Files containing private keys SHOULD only be readable by the entities that need them. When a private key is transported over an untrusted channel or stored in an untrusted location, it SHOULD be encrypted, for example using JSON Web Encryption (JWE) {{RFC7516}}.

Producers MUST take care not to accidentally publish a private DNSSEC JWK when a public key was intended. Consumers that only need the public key SHOULD discard any private key parameters.

The "dnskey" parameter is not cryptographically bound to the key material. An attacker able to modify a DNSSEC JWK could change, for example, the owner name or flags, causing the key to be used for a different zone or in a different role than intended. The key tag check in the processing rules (when the "keytag" member is present) detects accidental inconsistencies, but not deliberate modification, since an attacker can recompute the key tag. The integrity of DNSSEC JWKs MUST therefore be ensured by other means, such as an authenticated transport or a digital signature over the key.

The security considerations of {{RFC7517}}, {{RFC7518}}, {{RFC8037}} and {{RFC4034}} also apply.

--- back

# Example Keys {#example-keys}

This appendix contains example private DNSSEC JWKs, each followed by the DNSKEY resource record it corresponds to. The keys were generated for this document and are not to be used for any other purpose.

Long lines are wrapped using the single backslash strategy described in {{RFC8792}}.

The owner name of the last example, "b\\000rken.example.com.", has a zero octet in its first label. In DNS presentation format this octet is written "\\000", and because the backslash is escaped again in JSON, it appears as "b\\\\000rken" in the serialized JWK.

## RSA/SHA-256

The following private key uses RSA/SHA-256 (algorithm 8, mnemonic RSASHA256), with owner name "example.com." and key tag 49458:

~~~ json
=============== NOTE: '\' line wrapping per RFC 8792 ================

{
  "kid": "jScRJOUR4SgZYG33jZLNS2rCnBVUweOV8vJczsU0eJ0",
  "kty": "RSA",
  "n": "xxNpapt2fBfaMG5rW1rJYRTT2lLRvh3MMlWaKNxW44Q494NSaT0b8sHFBrkn\
OqWjA1cRoeHVRdoartG6IlZlZnnCju6FhXb1mWk6IpSlj8fUNjrowPqTbGyw2FA00j3n\
l0UJbhwwDp9HiFj-uAm0pX_oMYmEEkTNHrjW4eb6REPg8-EboU5PcDZx_HHiCPVkyc4C\
XKRf5ZEBhYnPIYcxuwei_rKj8QOheszdHDcbw6ysg9GG8yB9Q3P0AvgwDoPevUTL18yJ\
u-DMaHWpSJ2H2tQ45Jqy0lmdF3dImQUipXswc51RM09cGZe8i3nH2GqjM-1GzfFtAdZ3\
M5plbCfndw",
  "e": "AQAB",
  "d": "A2xPIG3ags-KlgRoCyxlKkRIUK8I4grmhnak9A8dEJENXunFD0zLzLsa1X7c\
ilK4mieZ0YQUSnAYJY__2KeBdQ28VABAKOdyDKvCO9ljfJsg7oSWMpWvo3ckL7jQiubu\
mWuVLTpsQLjp0XhHFPuk8NphfNL5A7MA9ezGurnhufrxaOf2dD6sKU9afePsnEfNS2Ex\
VmgGcJE5WzFnso1s0tAROWj2SkTJwSENqEx2vjSMySJAWBkG7otL5NDm4xXkuALrxx1C\
CTVJR5y5SGypSZOYDE7VFj-E9KE2E2L7aReZ7omoYXTWvgPy6XO4Y9pDR8Sq0W7O9yTs\
gvUe_u_xYQ",
  "p": "-3yQvGcP2er34X2QGI99uUoALS0E1SQNEny-xFP-QM82ootCTTEQMkT399BH\
l-d4DKPLpxssCLUm-KVntR4NetqDSWEqS9h3gsrkT8g6luhcFTJWnsYgiWTvFZwqMT_j\
ofRFsgvpuEDN_SeGXAFLpl3APMV1Qx6eF4vKcG1f-CE",
  "q": "yqYMqT0IO1Kx_XXEzLjSqG_Ew8MclvTuvp7CEjITedoNvTaVJAPs-jxN4_ea\
xT9Mrx88iXYvyS8WKB2SLu4x9wvu4NhmgcCUrh3fqCkBkpgU81qPLrjfRmnpp5h0cZK0\
puMHFcejv8bSH8Gstz6tPDXlNeZnOAF1j6Nxds-LDJc",
  "dp": "b_UsmjNnoyQv5nr3w3deCTcgjPwVwq2iBDD03IJoFoIpWIOWqLbG9xotKm3\
W8ZFiqH3jFCnMmLPM_bTlToRqi97rw6c2JT3hNqEMs1f4fWZEWEtotJaPtZrxtTUos3Q\
16svMm08iZZy0F5eZASyyeQYTpvphEdh3mRPqs9J9fwE",
  "dq": "HGtzJRzpFaDAU4J7qhZVfMfH-YhYbxdEkDklkHWzY4hgGuQXwGhIiMOCTPF\
BpF3_PCQOUlzf0MAoxFF5WdmP6kyjIKdYoOPINi78WdH-ZXesoIBXyj6lGXPrD9UCp5b\
ifPEj4jsQTOnuOAPqW_yPQP1x-sAoQ6N4wD80v2vV3Vs",
  "qi": "M-N7jYUpFWAqAUYgC7eZDGi0Pab-fpY01GvLxhYmclq1SdXpG4wOPIozXOd\
dRpMqYTKKgFksBsQRb2hNXkUxrHrNx5FzsUpzDk7F4RqfXjS-lT4ASO1BTeDdr_FVoLM\
8UnWHlFxmC5ZkUiL0W31i4TA9PMwGmjOaYCW213C6Gm0",
  "dnskey": {
    "name": "example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 8,
    "keytag": 49458
  }
}
~~~
{: title="RSA/SHA-256 Private Key"}

The corresponding DNSKEY resource record is:

~~~ dns-rr
example.com. IN DNSKEY 256 3 8 (
    AwEAAccTaWqbdnwX2jBua1tayWEU09pS0b4dzDJVmijcVuOEOPeDUmk9
    G/LBxQa5JzqlowNXEaHh1UXaGq7RuiJWZWZ5wo7uhYV29ZlpOiKUpY/H
    1DY66MD6k2xssNhQNNI955dFCW4cMA6fR4hY/rgJtKV/6DGJhBJEzR64
    1uHm+kRD4PPhG6FOT3A2cfxx4gj1ZMnOAlykX+WRAYWJzyGHMbsHov6y
    o/EDoXrM3Rw3G8OsrIPRhvMgfUNz9AL4MA6D3r1Ey9fMibvgzGh1qUid
    h9rUOOSastJZnRd3SJkFIqV7MHOdUTNPXBmXvIt5x9hqozPtRs3xbQHW
    dzOaZWwn53c= )
~~~
{: title="RSA/SHA-256 DNSKEY Resource Record"}

## ECDSA Curve P-256 with SHA-256 {#example-ecdsap256sha256}

The following private key uses ECDSA Curve P-256 with SHA-256 (algorithm 13, mnemonic ECDSAP256SHA256), with owner name "example.com." and key tag 9115:

~~~ json
{
  "kid": "ni1azkx1Qe3zOuUw1eRd3oNN_91av68ll49pGTLF_BM",
  "kty": "EC",
  "crv": "P-256",
  "x": "9cqheie08FXlvnU17YJJm1FMUXD4rwi7-4atSdZ3og8",
  "y": "PR66xkQK4YKumafBX4tYV4N008RfZHQZdMxs_7ToK5A",
  "d": "ZaTVn0mC1OgwzCvuQ1V3qf6z-6mMjDStD2QzzUQoHxA",
  "dnskey": {
    "name": "example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 13,
    "keytag": 9115
  }
}
~~~
{: title="ECDSA Curve P-256 with SHA-256 Private Key"}

The corresponding DNSKEY resource record is:

~~~ dns-rr
example.com. IN DNSKEY 256 3 13 (
    9cqheie08FXlvnU17YJJm1FMUXD4rwi7+4atSdZ3og89HrrGRArhgq6Z
    p8Ffi1hXg3TTxF9kdBl0zGz/tOgrkA== )
~~~
{: title="ECDSA Curve P-256 with SHA-256 DNSKEY Resource Record"}

## Ed25519

The following private key uses Ed25519 (algorithm 15, mnemonic ED25519), with owner name "example.com." and key tag 63713:

~~~ json
{
  "kid": "sddEIwsSByQMb2_YrSxRWGFZoIZSxVP9SWB7_IZ4X5Q",
  "kty": "OKP",
  "crv": "Ed25519",
  "x": "xX2gT6qJ76coFM3_pAUBxfBZYGnfSZqbNvbZFI428Ao",
  "d": "Dz9KOncZ00D1WiPEJR_Ec_WXR7YuCbdhUO_H9AERYvw",
  "dnskey": {
    "name": "example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 15,
    "keytag": 63713
  }
}
~~~
{: title="Ed25519 Private Key"}

The corresponding DNSKEY resource record is:

~~~ dns-rr
example.com. IN DNSKEY 256 3 15 (
    xX2gT6qJ76coFM3/pAUBxfBZYGnfSZqbNvbZFI428Ao= )
~~~
{: title="Ed25519 DNSKEY Resource Record"}

## ML-DSA-44

The following private key uses ML-DSA-44 (algorithm 18, mnemonic MLDSA44), with owner name "example.com." and key tag 59828:

~~~ json
=============== NOTE: '\' line wrapping per RFC 8792 ================

{
  "kid": "XqS8KAWB0Sc42bmWeKxan98eQKcHS4DVOHGc_kr_68E",
  "kty": "AKP",
  "alg": "ML-DSA-44",
  "pub": "17K0clSq4NtF55MNSpjSyX2PE5fReJ2voXAksxbpvslPyZRtQvGbeadBO7\
qjPnFJy0LtURVpOsBB-suYit61_g4dhjEYSZW1ksOX0ilOLhT5CqQUujgmiZrEP0zMrL\
wm6agyuVEY1ctDPL75ZgsAE44IF_YediyidMNq1VTrIqrBFi5KsBrLoeOMTv2PgLZbMz\
0PcuVd_nHOnB67mInnxWEGwP1zgDoq7P6v3teqPLLO2lTRK9jNNqeM-XWUO0er0l6ICs\
RS5XQu0ejRqCr6huWQx1jBWuTShA2SvKGlCQ9ASWWX_KfYuVE_GhvabpUKqpjeRnUH1K\
T1pPBZkhZYLDVy9i7aiQWrNYFnDEoCd3oz4Mpylf2PT_bRoKOnaD1l9fX3_GDaAj6CbF\
-SFEwC99G6EHWYdVPqk2f8122ZC3-pnNRa_biDbUPkWfUYffBYR5cJoB6mg1k1-nBGCZ\
DNPcG6QBupS6sd3kGsZ6szGdysoGBI1MTu8n7hOpwX0FOPQw8tZC3CQVZg3niHfY2KvH\
JSOXjAQuQoX0MZhGxEEmJCl2hEwQ5Va6IVtacZ5Z0MayqW05hZBx_cws3nUkp77a5U6F\
sxjoVOj-Ky8-36yXGRKCcKr9HlBEw6T9r9n_MfkHhLjo5FlhRKDa9YZRHT2ZYrnqla8Z\
e05fxg8rHtFd46W-9fib3HnZEFHZsoFudPpUUx79wcvnTUSIV_R2vNWPIcC2U7O3ak4H\
amVZowJxhVXMY_dIWaq6uSXwI4YcqM0Pe62yhx9n1VMm10URNa1F9KG6aRGPuyyKMO7J\
OS7z-XcGbJrdXHEMxkexUU0hfZWMcBfD6Q_SDATmdLkEhuk3CjGgAdMvRzl55JBnSefk\
d_oLdFCPil8jeDErg8Jb04jKCw__dHi69CtxZn7arJfEaxKWQ-WG5bBVoMIRlG1PNuZ1\
vtWGD6BCoxXZgmFk1qkjfDWl-_SVSQpb1N8ki5XEqud4S2BWcxZqxCRbW0sIKgnpMj5i\
8geMW3Z4NEbe_XNq06NwLUmwiYRJAKYYMzl7xEGbMNepegs4fBkRR0xNQbU-Mql3rLbw\
6nXbZbs55Z5wHnaVfe9vLURVnDGncSK1IE47XCGfFoixTtC8C4AbPm6C3NQ-nA6fQXRM\
2YFb0byIINi7Ej8E-s0bG2hd1aKxuNu_PtkzZw8JWhgLTxktCLELj6u9_MKyRRjjLuoK\
XgyQTKhEeACD87DNLQuLavZ7w1W5SUAl3HsKePqA46Lb_rUTKIUdYHgZjpSTZRrnh-wC\
UfkiujDp9R32Km1yeEzz3SBTkxdt-jJKUSvZSXCjbdNKUUqGeR8Os28BRbCatkZRtKAx\
OymWEaKhxIiRYnWYdooxFAYLpEQ0ht9RUioc6IswmFwhb45u0XjdVnswSg1Mr7qIKig0\
LxepqiauWNtjAIPSw1j99WbD9dYqQoVnvJ6ozpXKoPNUdLC_qPM5olCrTfzyCDvo7vvB\
BV4Y_hU3DuyyYFZtg_8GshGq7EPKKbVMzQD4gVokZe8LRlFcx-QfMSTwnv_3OTCatYsp\
oUWaALzlA46TjJZ49y6w5O5f2q5m2fhXP8l_xCtJWfS_i2HXhDPoawM11ukZHE2L9Iez\
kFwQjP1qwksM633LfPUfhNDtaHuV6uscUzwG8NlwI9kqcIJYN7Wbpst9TlawqHwgOGKu\
jzFbpZJejt76Z5NpoiAnZhUfFqll-fgeznbMBwtVhp5NuXhM8FyDCzJCyDEg",
  "priv": "AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8",
  "dnskey": {
    "name": "example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 18,
    "keytag": 59828
  }
}
~~~
{: title="ML-DSA-44 Private Key"}

The corresponding DNSKEY resource record is:

~~~ dns-rr
example.com. IN DNSKEY 256 3 18 (
    17K0clSq4NtF55MNSpjSyX2PE5fReJ2voXAksxbpvslPyZRtQvGbeadB
    O7qjPnFJy0LtURVpOsBB+suYit61/g4dhjEYSZW1ksOX0ilOLhT5CqQU
    ujgmiZrEP0zMrLwm6agyuVEY1ctDPL75ZgsAE44IF/YediyidMNq1VTr
    IqrBFi5KsBrLoeOMTv2PgLZbMz0PcuVd/nHOnB67mInnxWEGwP1zgDoq
    7P6v3teqPLLO2lTRK9jNNqeM+XWUO0er0l6ICsRS5XQu0ejRqCr6huWQ
    x1jBWuTShA2SvKGlCQ9ASWWX/KfYuVE/GhvabpUKqpjeRnUH1KT1pPBZ
    khZYLDVy9i7aiQWrNYFnDEoCd3oz4Mpylf2PT/bRoKOnaD1l9fX3/GDa
    Aj6CbF+SFEwC99G6EHWYdVPqk2f8122ZC3+pnNRa/biDbUPkWfUYffBY
    R5cJoB6mg1k1+nBGCZDNPcG6QBupS6sd3kGsZ6szGdysoGBI1MTu8n7h
    OpwX0FOPQw8tZC3CQVZg3niHfY2KvHJSOXjAQuQoX0MZhGxEEmJCl2hE
    wQ5Va6IVtacZ5Z0MayqW05hZBx/cws3nUkp77a5U6FsxjoVOj+Ky8+36
    yXGRKCcKr9HlBEw6T9r9n/MfkHhLjo5FlhRKDa9YZRHT2ZYrnqla8Ze0
    5fxg8rHtFd46W+9fib3HnZEFHZsoFudPpUUx79wcvnTUSIV/R2vNWPIc
    C2U7O3ak4HamVZowJxhVXMY/dIWaq6uSXwI4YcqM0Pe62yhx9n1VMm10
    URNa1F9KG6aRGPuyyKMO7JOS7z+XcGbJrdXHEMxkexUU0hfZWMcBfD6Q
    /SDATmdLkEhuk3CjGgAdMvRzl55JBnSefkd/oLdFCPil8jeDErg8Jb04
    jKCw//dHi69CtxZn7arJfEaxKWQ+WG5bBVoMIRlG1PNuZ1vtWGD6BCox
    XZgmFk1qkjfDWl+/SVSQpb1N8ki5XEqud4S2BWcxZqxCRbW0sIKgnpMj
    5i8geMW3Z4NEbe/XNq06NwLUmwiYRJAKYYMzl7xEGbMNepegs4fBkRR0
    xNQbU+Mql3rLbw6nXbZbs55Z5wHnaVfe9vLURVnDGncSK1IE47XCGfFo
    ixTtC8C4AbPm6C3NQ+nA6fQXRM2YFb0byIINi7Ej8E+s0bG2hd1aKxuN
    u/PtkzZw8JWhgLTxktCLELj6u9/MKyRRjjLuoKXgyQTKhEeACD87DNLQ
    uLavZ7w1W5SUAl3HsKePqA46Lb/rUTKIUdYHgZjpSTZRrnh+wCUfkiuj
    Dp9R32Km1yeEzz3SBTkxdt+jJKUSvZSXCjbdNKUUqGeR8Os28BRbCatk
    ZRtKAxOymWEaKhxIiRYnWYdooxFAYLpEQ0ht9RUioc6IswmFwhb45u0X
    jdVnswSg1Mr7qIKig0LxepqiauWNtjAIPSw1j99WbD9dYqQoVnvJ6ozp
    XKoPNUdLC/qPM5olCrTfzyCDvo7vvBBV4Y/hU3DuyyYFZtg/8GshGq7E
    PKKbVMzQD4gVokZe8LRlFcx+QfMSTwnv/3OTCatYspoUWaALzlA46TjJ
    Z49y6w5O5f2q5m2fhXP8l/xCtJWfS/i2HXhDPoawM11ukZHE2L9IezkF
    wQjP1qwksM633LfPUfhNDtaHuV6uscUzwG8NlwI9kqcIJYN7Wbpst9Tl
    awqHwgOGKujzFbpZJejt76Z5NpoiAnZhUfFqll+fgeznbMBwtVhp5NuX
    hM8FyDCzJCyDEg== )
~~~
{: title="ML-DSA-44 DNSKEY Resource Record"}

## ED25519 with Escaped Owner Name

The following private key uses Ed25519 (algorithm 15, mnemonic ED25519), with owner name "b\\000rken.example.com." and key tag 17078:

~~~ json
{
  "kid": "uJYXQ0ZhGzTiAt0jgXgPDWKExUB3cls1NXInBRsqYU0",
  "kty": "OKP",
  "crv": "Ed25519",
  "x": "w_VxpL9dTguTG_ivgqZFa9lkjiPkR-g15ktPAhuTIt8",
  "d": "NQ4G7dnYiYJMO2UYGqG82Rn2zsX24P6lfiteOBR78PA",
  "dnskey": {
    "name": "b\\000rken.example.com.",
    "flags": 256,
    "protocol": 3,
    "algorithm": 15,
    "keytag": 17078
  }
}
~~~
{: title="ED25519 with Escaped Owner Name Private Key"}

The corresponding DNSKEY resource record is:

~~~ dns-rr
b\000rken.example.com. IN DNSKEY 256 3 15 (
    w/VxpL9dTguTG/ivgqZFa9lkjiPkR+g15ktPAhuTIt8= )
~~~
{: title="ED25519 with Escaped Owner Name DNSKEY Resource Record"}

# Acknowledgements
{:numbered="false"}

TBD.

