---
theme : "league"
transition: "slide"
highlightTheme: "none"
---

## Formal Verification of Gnosis Safe

<small>

Dominik Teiml

dominik@gnosis.pm

[dteiml.github.io/presentations](dteiml.github.io/presentations) --> Formal Verification of Gnosis Safe

</small>

---

## Gnosis Safe Contracts

--

## EIP 191



## EIP 712

This EIP aims to improve the usability of off-chain message signing for use on-chain. We are seeing growing adoption of off-chain message signing as it saves gas and reduces the number of transactions on the blockchain. Currently signed messages are an opaque hex string displayed to the user with little context about the items that make up the message.

Here we outline a scheme to encode data along with its structure which allows it to be displayed to the user for verification when signing. Below is an example of what a user could be shown when signing an EIP712 message.

The set of signable messages is extended from transactions and bytestrings $\mathbb{T} \cup \mathbb{B}^{8n}$ to also include structured data $\mathbb{S}$.

encode(domainSeparator : 𝔹²⁵⁶, message : 𝕊) = "\x19\x01" ‖ domainSeparator ‖ hashStruct(message) where domainSeparator and hashStruct(message)



---

<!-- .slide: style="text-align: left;" -->
## THE END