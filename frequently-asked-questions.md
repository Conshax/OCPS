# Frequently Asked Questions

<details>

<summary>What if my audio library doesn’t let me handle 402 responses?</summary>

For our test implementation, we extended the `podcast:valueRecipient` tag with a `lsat-keysend` attribute. This attribute can be used to identify the payment verifier by clients that don’t want / can’t parse data from the 402 response:&#x20;

```xml
<podcast:valueRecipient 
  name=“Conshax” 
  address=“02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca” 
  type=“node” 
  split="10" 
  lsat-keysend=“true” 
 />
```

That way, you can pay for the episode in advance and request the episode with the preimage without ever receiving a 402 response from the server. See [Why don’t you introduce new XML tags for the RSS feed?](frequently-asked-questions.md#why-dont-you-introduce-new-xml-tags-for-the-rss-feed) to understand why we don't include that in the protocol for now.

</details>

<details>

<summary>Why don't you introduce new XML tags for the RSS feed?</summary>

First and foremost, we want to keep the integration workload for clients and servers as little as possible. A main strength of LSAT is that it only relies on the HTTP protocol, which makes it easy to integrate. We want to maintain this strength.

In the future, it could be useful to introduce new XML tags or attributes to transport meta information if clients and servers find this to be useful.&#x20;

For example, we extended the `podcast:valueRecipient` tag with a `lsat-keysend` attribute in our example podcast. This attribute can be used to identify the payment verifier by clients that don't want / can't parse data from the 402 response:

```rss
<podcast:valueRecipient 
  name="Conshax" 
  address="02b5f5a96d6c0cfb7ad6adda59c25eba3c12a9a0beee22a8b31d3d20b59427bbca" 
  type="node" 
  split="10" 
  lsat-keysend="true"
 />
```

</details>

<details>

<summary>What if the RSS feed contains multiple splits for the payment verifier?</summary>

The server responds with the address (+ customKey & customValue if necessary) to identify the payment verifier. If there are multiple splits for the payment verifier, the client can simply choose the preimage of the payment with the largest split.

</details>

<details>

<summary>Why do you use keysend payments, not invoices?</summary>

This design decision is driven by the popularity of [Value4Value](https://value4value.info) payments in [Podcasting 2.0](https://github.com/Podcastindex-org/podcast-namespace/blob/main/docs/1.0.md), which rely on value tags as well as keysend payments and are already integrated by a number of popular podcast players such as Breez, Fountain, Podverse and Castamatic. The usage of keysend payments eases the LSAT integration and ensures compatibility with Value4Value.

However, OCPS is designed to be extensible so invoices could be used in the future.

</details>

<details>

<summary>What is the use of the token contained in 402 responses?</summary>

The token contains arbitrary data that the server uses to associate a request to a client.

If a server has no reason to associate a request to a client, e.g., to restrict access by time or geographic region, it can simply return a constant for all requests.

</details>

<details>

<summary>Can I use this protocol to securely distribute paid audio content?</summary>

When a server implements the OCPS to protect resources, it securely verifies that only clients with a valid payment preimage gain access.

The payment preimage proves that a valid payment to the payment verifier (as chosen by the server) was made by the client. Payments to other recipients defined in the podcast's RSS feed are not verified by default. However, the server could validate these additionally if it has read access to the recipients' nodes.

The server can also use the `token` that he sends to the client in the 402 response to introduce additional security measures, e.g., restricting resource access by time and geographic region.

However, as in every other protocol that distributes digital resources, it is not possible to completely prevent malicious client behavior, e.g., sharing a valid preimage & token with other clients or duplicating the resource after rightfully gaining access to it.

Still, the OCPS provides a much more secure approach to protecting resources in comparison to most existing solutions because it protects each audio file in a podcast feed individually.

</details>
