---
description: >-
  This page specifies how clients (that request a resource) and servers (that
  provide a resource) interact with each other.
---

# Protocol Specification

## Introduction

The following sections provide a fine-grained explanation of the protocol that can be used as a template for clients and hosts that want to support the OCPS.

If you are just looking for a quick way to get started, refer to the [Example Implementation](example-implementation.md) page instead.

## Protocol Overview

<figure><img src=".gitbook/assets/mermaid-diagram-2023-02-11-120912.png" alt=""><figcaption></figcaption></figure>

### **Client without LSAT capability**

When a client that does not support LSAT request a protected resource, the server simply responds with the free part of the protected audio file. This is the case for both download and streaming requests.

****

### **Client with LSAT capability**

**1. Client requests a protected resource**

A client tells the server that it supports LSAT by setting the `X-Accept-Authenticate` header.&#x20;

{% tabs %}
{% tab title="Recommended: Header" %}
```http
GET /protected HTTP/2
X-Accept-Authenticate: lsat-keysend
```
{% endtab %}

{% tab title="Alternative: Query parameter" %}
If the client is unable to set custom headers, it can set the information as a query parameter.

```http
GET /protected?lsat-accept-authenticate=lsat-keysend HTTP/2
```
{% endtab %}
{% endtabs %}

The server responds with status code 402 and communicates the payment information in the `WWW-Authenticate` header. This information identifies a unique recipient, the _**payment verifier**_, and a minimum amount that must be sent to that recipient to access the resource.

{% tabs %}
{% tab title="Recommended: Header" %}
<pre class="language-http"><code class="lang-http">HTTP/2 402 Payment Required
<strong>WWW-Authenticate: lsat-keysend token="abc", address="02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca", custom_key="696969", custom_value="4VqhBQ73TSgpTFbJ35C3", amount="10"
</strong># length of content that is available for free
<strong>Lsat-Free-Content-Length: 500000
</strong></code></pre>

See [streams vs download](protocol-specification.md#differences-between-streams-and-downloads) for more information on `Lsat-Free-Content-Length`.
{% endtab %}

{% tab title="Alternative: Response body" %}
To support clients that cannot read this header, the server also returns this information in the response body.

```json
{
    "token": "abc",
    "address": "02b5.....", // payment verifier
    "custom_key": "696969", //optional
    "custom_value": "4VqhBQ73TSgpTFbJ35C3", //optional
    "amount": 10 //amount to pay in sats
    
}
```
{% endtab %}
{% endtabs %}



**2. Client identifies additional payment recipients and conducts payments**\
\
A podcast episode can have multiple recipients that are defined in the episode's `podcast:value` tag. The _**payment verifier**_ (from step 1) is always listed as a value recipient in the RSS feed:

```xml
<podcast:value type="lightning" method="keysend" suggested="0.00000100">
  <podcast:valueRecipient name="Conshax" address="02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca" type="node" split="10" customKey="696969" customValue="4VqhBQ73TSgpTFbJ35C3" fee="true"/>
  <podcast:valueRecipient name="Test user" address="030a58b8653d32b99200a2334cfe913e51dc7d155aa0116c176657a4f1722677a3" type="node" split="90" customKey="696969" customValue="4VqhBQ73TSgpTFbJ35C3" fee="false"/>
</podcast:value> 
```

The client uses the `suggested` attribute as the (minimum) total amount, calculates the individual amounts and makes one keysend payment per `valueRecipient`. The payments must at least include the `episode_guid` property to make the payment relatable to the episode ([blip-10](https://github.com/lightning/blips/blob/master/blip-0010.md)). We recommend to provide all known blip-10 properties, to set the `action` to “lsat", and to set a `token` property to the value of the token provided by the server (see step 1).

The client stores the preimage of the keysend payment to the _**payment verifier**_ (from step 1) in a local variable.\


**3. Client requests full resource using payment preimage**

The client requests the full resource by setting the hex representation of the keysend payment preimage to the _**payment verifier**_** ** (see step 1) and the token (also received in step 1) in the `Authorization` header.\
If the client is unable to set custom headers, it can set the information as a URL encoded query parameter.

{% tabs %}
{% tab title="Recommended: Header" %}
```http
GET /protected HTTP/2
# LSAT <token>:<payment-preimage>
Authorization: LSAT abc:1234abcd1234abcd1234abc 
```
{% endtab %}

{% tab title="Alternative: Query parameter" %}
```http
GET /protected?lsat-authorization=abc%3A1234abcd1234abcd1234abc HTTP/2 
```
{% endtab %}
{% endtabs %}

The server uses the preimage to verify that the authenticating payment was made and is related to the requested resource. If that is the case, the server returns the full resource:&#x20;

```http
HTTP/2 200 Ok
Content-Type: audio/mpeg
Content-Length: 15712231

<full audio file>
```



## Differences between streams & downloads

When a client wants to stream an audio file, it doesn't want to download the whole file before it can start playing. Instead, the client repeatedly queries a specific part of the audio file using the `Range` header.&#x20;

The `Range` **must** be set via header. OCPS specific parameters can still be set via header or query parameter in range requests.&#x20;

Stream request spec:

{% tabs %}
{% tab title="Free range" %}
Client request:

```http
GET /protected HTTP/2 
# also possible via url: /protected?lsat-accept-authenticate=lsat-keysend
X-Accept-Authenticate: lsat-keysend
Range: 0-50000
```

Server response:

```http
HTTP/2 206 OK
Content-Type: audio/mpeg
Content-Range: bytes 0-50000/15712231
Content-Length: 50001
# length of content that is available for free
Lsat-Free-Content-Length: 5000000
Accept-Ranges: bytes
```
{% endtab %}

{% tab title="Protected range (before payment)" %}
Client request:

```http
GET /protected HTTP/2 
# also possible via url: /protected?lsat-accept-authenticate=lsat-keysend
X-Accept-Authenticate: lsat-keysend
Range: 500001-550000
```

Server response:

```http
HTTP/2 402 Payment Required
# also available in response body
WWW-Authenticate: lsat-keysend token=“abc”, address=“02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca”, custom_key=“696969", custom_value=“4VqhBQ73TSgpTFbJ35C3”, amount=“10”
# length of content that is available for free
Lsat-Free-Content-Length: 500000
```
{% endtab %}

{% tab title="Protected range (after payment)" %}
Client request:

```http
GET /protected HTTP/2 
# also possible via url: /protected?lsat-authorization=abc%3A1234abcd1234abcd1234ab
Authorization: LSAT abc:1234abcd1234abcd1234abc
Range: 500001-550000
```

Server response:

```http
HTTP/2 206 OK
Content-Type: audio/mpeg
Content-Range: bytes 550001-550000/15712231
Content-Length: 50000
Accept-Ranges: bytes
```
{% endtab %}
{% endtabs %}







