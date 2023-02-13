# Example Implementation

## Disclaimer

The Postman desktop app currently crashes on responses with a binary in the body (see this [issue](https://github.com/postmanlabs/postman-app-support/issues/11629)). We recommend to use the web app or our [curl examples](example-implementation.md#full-interaction-examples-shell-curl) for testing.

## Test Podcast

We have created a test podcast, "The OCPS Show", for testing and prototyping purposes that contains one episode. This episode's audio file behaves adherent to the OCPS.

{% tabs %}
{% tab title="Test Podcast" %}
**Name:** "The OCPS Show"

**Link:** [https://podcastindex.org/podcast/6004846](https://podcastindex.org/podcast/6004846)

**Feed URL:** \
[https://podcasts-public.s3.eu-central-1.amazonaws.com/a3a1b651-94e2-56f0-9dee-a23813aa7220/feed.rss](https://podcasts-public.s3.eu-central-1.amazonaws.com/a3a1b651-94e2-56f0-9dee-a23813aa7220/feed.rss)

**Episode audio URL:** \
[https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3](https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3)

**Episode Length:** 03:30 / 10:54 (free / full)

**Episode GUID:** `a3a1b651-94e2-56f0-9dee-a23813aa7220`
{% endtab %}
{% endtabs %}

### Test episode behavior

* The first 03:30 minutes of the episode are available for free
* Download request:
  * <mark style="background-color:orange;">No LSAT support:</mark> 03:30 minutes are downloaded
  * <mark style="background-color:green;">LSAT support:</mark> server responds with status 402 "payment required" and the necessary payment information
* Streaming request:
  * <mark style="background-color:orange;">No LSAT support:</mark> client can stream 03:30 minutes, then the episode is "finished"
  * <mark style="background-color:green;">LSAT support:</mark> client can stream 03:30 minutes, then the the server responds with 402 "payment required" and the necessary payment information



## Interacting with the test episode

### Query file information

{% swagger method="head" path=" " baseUrl="{episode audio url}  " summary="Retrieve file meta data" %}
{% swagger-description %}
Query HTTP headers to learn meta data about the file, e.g., the file size.
{% endswagger-description %}

{% swagger-parameter in="header" name="X-Accept-Authenticate" %}
lsat-keysend (optional)
{% endswagger-parameter %}

{% swagger-response status="200: OK" description="" %}
{% tabs %}
{% tab title="Without header" %}
```http
HTTP/2 200 Ok
Content-Type: audio/mpeg
Content-Length: 5000000
Accept-Ranges: bytes
```

The `Accept-Ranges` header indicates that the client can request parts of the resource with custom byte ranges.
{% endtab %}

{% tab title="With header" %}
```http
HTTP/2 200 Ok
Content-Type: audio/mpeg
Content-Length: 15712231
Accept-Ranges: bytes
Lsat-Free-Content-Length: 5000000
WWW-Authenticate: lsat-keysend token="AAAA", address="02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca", custom_key="696969", custom_value="4VqhBQ73TSgpTFbJ35C3", amount="10"
```

The `Lsat-Free-Content-Length` header specifies the **maximum number of bytes** that can be requested without payment.

The `WWW-Authenticate` header contains the information for the payment required to access the full resource.


{% endtab %}
{% endtabs %}
{% endswagger-response %}
{% endswagger %}

###

### Make a valid keysend payment

For testing purposes, it is sufficient to pay the recipient returned in the 402 response. The SATS you send during testing go to our node, we'll happily return them to you if you contact us.

Make sure the keysend payment contains the following data in the custom records:

```json
{
    "episode_guid": "a3a1b651-94e2-56f0-9dee-a23813aa7220", //required
    "token": "AAAA", //recommended
    "action": "lsat" // recommended
}
```

If you can't or don't want to make the keysend payment, you can use this preimage for testing:

`3630626264383464316435666130356136653430616139386337663331643365`

###

### Download the episode audio

<mark style="background-color:orange;">Client with no LSAT support:</mark>

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Download request (no LSAT)" expanded="false" %}
{% swagger-description %}

{% endswagger-description %}

{% swagger-response status="200: OK" description="Audio file" %}


```http
HTTP/2 200 Ok
Content-Type: audio/mpeg
Content-Length: 5000000

<free audio file>
```
{% endswagger-response %}
{% endswagger %}

<mark style="background-color:green;">Client with LSAT support:</mark>

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Download request before payment" expanded="false" %}
{% swagger-description %}
LSAT support is communicated by setting either the header **OR** the query parameter.

**Hint:** Keep in mind that query parameters are case sensitive.
{% endswagger-description %}

{% swagger-parameter in="query" name="lsat-accept-authenticate" type="" %}
lsat-keysend
{% endswagger-parameter %}

{% swagger-parameter in="header" name="X-Accept-Authenticate" type="" %}
lsat-keysend
{% endswagger-parameter %}

{% swagger-response status="402: Payment Required" description="Payment required" %}
The payment information can be retrieved from the body or the `WWW-Authenticate` header:

```http
HTTP/2 402 Payment Required
Content-Type: application/json
WWW-Authenticate: lsat-keysend token="AAAA", address="02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca", custom_key="696969", custom_value="4VqhBQ73TSgpTFbJ35C3", amount="10"

{
  "error": "Payment Required",
  "address": "02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca",
  "amount": 10,
  "custom_key": "696969",
  "custom_value": "4VqhBQ73TSgpTFbJ35C3",
  "token": "AAAA"
}
```
{% endswagger-response %}
{% endswagger %}

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Download request after payment" expanded="false" %}
{% swagger-description %}
The token (provided by the server in the 402 return) and the hex encoded payment preimage can be set via header **OR** query parameter.

**Hint:** Keep in mind that query parameters are case sensitive and must be URL encoded.
{% endswagger-description %}

{% swagger-parameter in="query" name="lsat-authorization" type="" %}
\<token:preimage> (URL encoded)
{% endswagger-parameter %}

{% swagger-parameter in="header" name="Authorization" type="" %}
\<token:preimage>
{% endswagger-parameter %}

{% swagger-response status="200: OK" description="Audio file" %}
```http
HTTP/2 200 Ok
Content-Type: audio/mpeg
Content-Length: 15712231

<full audio file>
```
{% endswagger-response %}

{% swagger-response status="400: Bad Request" description="Bad request" %}
Server responds with 400 as well as an error description and the payment information in the body:

```http
HTTP/2 400 Bad request
Content-Type: application/json

{
    "error": "a description of the client error",
    "token": "abc",
    "address": "02b5.....",
    "custom_key": "696969",
    "custom_value": "ABC",
    "amount": 10
}
```
{% endswagger-response %}
{% endswagger %}



### Stream the episode audio

<mark style="background-color:orange;">Client with no LSAT support:</mark>

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Stream request (no LSAT)" expanded="false" %}
{% swagger-description %}

{% endswagger-description %}

{% swagger-parameter in="header" name="Range" required="true" %}
e.g.: bytes=0-40000
{% endswagger-parameter %}

{% swagger-response status="206: Partial Content" description="Partial audio file" %}


```http
HTTP/2 206 Partial Content
Content-Type: audio/mpeg
Content-Range: bytes 0-39999/500000
Content-Length: 40000
Accept-Ranges: bytes

<partial audio file>
```
{% endswagger-response %}
{% endswagger %}

<mark style="background-color:green;">Client with LSAT support:</mark>

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Stream request before payment" expanded="false" %}
{% swagger-description %}
LSAT support is communicated by setting either the header **OR** the query parameter.

**Hint:** Keep in mind that query parameters are case sensitive.
{% endswagger-description %}

{% swagger-parameter in="query" name="lsat-accept-authenticate" type="" %}
lsat-keysend
{% endswagger-parameter %}

{% swagger-parameter in="header" name="X-Accept-Authenticate" type="" %}
lsat-keysend
{% endswagger-parameter %}

{% swagger-parameter in="header" name="Range" required="true" %}
e.g.: bytes=0-40000
{% endswagger-parameter %}

{% swagger-response status="206: Partial Content" description="Partial audio file" %}
**Requested byte range is freely available:**

```http
HTTP/2 206 Partial Content
Content-Type: audio/mpeg
Content-Range: bytes 0-39999/15712231
Content-Length: 40000
Lsat-Free-Content-Length: 500000
Accept-Ranges: bytes

<partial audio file>
```
{% endswagger-response %}

{% swagger-response status="402: Payment Required" description="Payment required" %}
**Requested byte range is protected.**

The payment information can be retrieved from the body or the `WWW-Authenticate` header:

```http
HTTP/2 402 Payment Required
Content-Type: application/json
WWW-Authenticate: lsat-keysend token=“AAAA”, address=“02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca”, custom_key=“696969", custom_value=“4VqhBQ73TSgpTFbJ35C3”, amount=“10”

{
    "token": "AAAA",
    "address": "02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca",
    "custom_key": "696969",
    "custom_value": "ABC",
    "amount": 10
}
```
{% endswagger-response %}
{% endswagger %}

{% swagger method="get" path="  " baseUrl="{episode audio url}" summary="Stream request after payment" expanded="false" %}
{% swagger-description %}
The token (provided by the server in the 402 return) and the hex encoded payment preimage can be set via header **OR** query parameter.

**Hint:** Keep in mind that query parameters are case sensitive and must be URL encoded.
{% endswagger-description %}

{% swagger-parameter in="query" name="Lsat-Authorization" type="" %}
\<token:preimage> (URL encoded)
{% endswagger-parameter %}

{% swagger-parameter in="header" name="Authorization" type="" %}
\<token:preimage>
{% endswagger-parameter %}

{% swagger-parameter in="header" name="Range" required="true" %}
e.g.: bytes=550001-560000
{% endswagger-parameter %}

{% swagger-response status="206: Partial Content" description="Partial audio file" %}
```http
HTTP/1.1 206 Partial Content
Content-Type: audio/mpeg
Content-Range: bytes 550001-560000/15712231
Content-Length: 50000
Accept-Ranges: bytes

<partial audio file>
```
{% endswagger-response %}

{% swagger-response status="400: Bad Request" description="Bad request" %}
Server responds with 400 as well as an error description and the payment information in the body:

```http
HTTP/1.1 400 Bad request
Content-Type: application/json

{
    "error": "a description of the client error",
    "token": "AAAA",
    "address": "02b5.....",
    "custom_key": "696969",
    "custom_value": "ABC",
    "amount": 10
}
```
{% endswagger-response %}
{% endswagger %}



## Full interaction examples (Shell / Curl)

{% tabs %}
{% tab title="No LSAT support" %}
```sh
#!/bin/sh

# head request to retrieve metadata 
echo "> file metadata:"
curl -I https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3

# range (stream) request for a part of the free content
echo "\n> this request will download the first part of the freely available content (lsat-test-free-first-part.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 -H "range: bytes=0-2500000" --output lsat-test-free-first-part.mp3

# download request for the full free resource
echo "\n> this request will download the freely available content (lsat-test-free.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 --output lsat-test-free.mp3
```
{% endtab %}

{% tab title="LSAT via headers" %}
```sh
#!/bin/sh

# head request to retrieve metadata 
echo "> file metadata:"
curl -I https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3  -H "X-Accept-Authenticate: lsat-keysend"

# range (stream) request for free range
echo "\n> this request will download the freely available content (lsat-test-free.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 -H "X-Accept-Authenticate: lsat-keysend" -H "range: bytes=0-5000000" --output lsat-test-free.mp3

# range (stream) request for protected range without payment
echo "\n> this request will fail with 402 because it requests protected content with an X-Accept-Authenticate header:"
curl -I -X GET https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 -H "X-Accept-Authenticate: lsat-keysend" -H "range: bytes=0-10000000"

# range (stream) request for protected range with payment
echo "\n> this request will download a part of the protected content using a valid preimage (lsat-test-more-than-free.mp3):"
curl -I -X GET https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 -H "AUTHORIZATION: LSAT AAAA:3630626264383464316435666130356136653430616139386337663331643365" -H "range: bytes=0-10000000" --output lsat-test-more-than-free.mp3

# download request with payment
echo "\n> this request will download the full resource using a valid preimage (lsat-test-full.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3 -H "AUTHORIZATION: LSAT AAAA:3630626264383464316435666130356136653430616139386337663331643365" --output lsat-test-full.mp3
```
{% endtab %}

{% tab title="LSAT via query parameters" %}
```sh
#!/bin/sh

# head request to retrieve metadata 
echo "> file metadata:"
curl -I https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3

# range (stream) request for free range
echo "\n> this request will download the freely available content (lsat-test-free.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3?lsat-accept-authenticate=lsat-keysend -H "range: bytes=0-5000000" --output lsat-test-free.mp3

# range (stream) request for protected range without payment
echo "\n> this request will fail with 402 because it requests protected content with an lsat-accept-authenticate query parameter:"
curl -I -X GET https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3?lsat-accept-authenticate=lsat-keysend -H "range: bytes=0-1000000"

# range (stream) request for protected range with payment
echo "\n> this request will download a part of the protected content using a valid preimage (lsat-test-more-than-free.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3?lsat-authorization=AAAA%3A3630626264383464316435666130356136653430616139386337663331643365 -H "range: bytes=0-10000000" --output lsat-test-more-than-free.mp3

# download request with payment
echo "\n> this request will download the full resource using a valid preimage (lsat-test-full.mp3):"
curl https://lsat-test.conshax.app/podcast/a3a1b651-94e2-56f0-9dee-a23813aa7220/episode/lsat-test.mp3?lsat-authorization=AAAA%3A3630626264383464316435666130356136653430616139386337663331643365 --output lsat-test-full.mp3
```
{% endtab %}
{% endtabs %}

