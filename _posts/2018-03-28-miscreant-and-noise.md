---
layout: post
title: "Introducing Miscreant.NET and Noise.NET"
date: 2018-03-28 19:50:00 +0200
---
![](/assets/img/2018-03-28-miscreant.svg)

One of the most common cryptographic tasks that programmers face
is data encryption. Modern symmetric cryptography is built around
[AEAD] (authenticated encryption with associated data) ciphers. If used
correctly (i.e. if you never reuse the same combination of the key and
the nonce), AEAD encryption modes (e.g. [AES-GCM] and [ChaCha20-Poly1305])
offer both confidentiality and integrity; otherwise, they often fail
[catastrophically]. To combat the nonce reuse problem, cryptographers
have come up with so called [nonce misuse resistant] schemes, where repeated
nonces don't lead to plaintext compromise. Perhaps the best-known such
algorithm is [AES-GCM-SIV].

Now that you are familiar with the basics of modern symmetric cryptography,
let's talk about what your options are if you are a .NET developer. Sadly,
the .NET standard library is lagging far behind the latest cryptographic trends.
It doesn't have any authenticated encryption mode. If you do a search for
"C# encryption", you will most likely find the article in the official
documentation called [Encrypting Data]. Its cipher of choice is unauthenticated
AES in CBC mode. It should go without saying that using such code in production
is incredibly dangerous.

So if the standard library is not the right tool for encrypting data, are there
at least some third-party libraries that offer what we need? [Bouncy Castle] is
sometimes recommended, but it's even more horrible choice than the standard
library and should be avoided. [libsodium-net] is a fine option, but it's not
compatible with the .NET Core. And we are still talking about AEADs only: as
far as I know, not a single one cryptographic library for .NET implements a
misuse resistant cipher.

[Miscreant] is a multi-language misuse resistant encryption library based on
a lesser-known, but much easier to implement and understand AES mode called
[AES-SIV]. AES-SIV was created by one of the most famous working cryptographers,
[Phil Rogaway]; the Miscreant library was originally developed by [Tony Arcieri]
for five different programming languages. I won't go into technical details here,
because everything you want to know about the library is already explained
in [this post] and the [Miscreant wiki].

I'm writing all this because I want to announce the [Miscreant.NET], which is
the .NET version of Miscreant, and the first ever .NET implementation of some
misuse resistant encryption scheme (it's actually a few months old—until now
I was just too lazy to write about it, but together with Noise.NET it makes a
perfect bundle). Code speaks louder than words, so here's one basic usage example:

[AEAD]: https://www.imperialviolet.org/2015/05/16/aeads.html
[AES-GCM]: https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf
[ChaCha20-Poly1305]: https://tools.ietf.org/html/rfc7539
[catastrophically]: https://www.usenix.org/system/files/conference/woot16/woot16-paper-bock.pdf
[nonce misuse resistant]: https://www.lvh.io/posts/nonce-misuse-resistance-101.html
[AES-GCM-SIV]: https://www.imperialviolet.org/2017/05/14/aesgcmsiv.html
[Miscreant]: https://miscreant.io/
[AES-SIV]: http://web.cs.ucdavis.edu/~rogaway/papers/keywrap.pdf
[Phil Rogaway]: http://web.cs.ucdavis.edu/~rogaway/
[Tony Arcieri]: https://tonyarcieri.com/
[this post]: https://tonyarcieri.com/introducing-miscreant-a-multi-language-misuse-resistant-encryption-library
[Miscreant wiki]: https://github.com/miscreant/miscreant/wiki
[Encrypting Data]: https://docs.microsoft.com/en-us/dotnet/standard/security/encrypting-data
[Bouncy Castle]: http://www.bouncycastle.org/csharp/
[libsodium-net]: https://github.com/adamcaudill/libsodium-net
[Miscreant.NET]: https://github.com/miscreant/miscreant/tree/master/dotnet

```csharp
// Plaintext to encrypt.
var plaintext = "I'm cooking MC's like a pound of bacon";

// Create a 32-byte key.
var key = Aead.GenerateKey256();

// Create a 16-byte nonce (optional).
var nonce = Aead.GenerateNonce(16);

// Create a new AEAD instance using the AES-CMAC-SIV
// algorithm. It implements the IDisposable interface,
// so it's best to create it inside using statement.
using (var aead = Aead.CreateAesCmacSiv(key))
{
  // If the message is string, convert it to byte array first.
  var bytes = Encoding.UTF8.GetBytes(plaintext);

  // Encrypt the message.
  var ciphertext = aead.Seal(bytes, nonce);

  // To decrypt the message, call the Open method with the
  // ciphertext and the same nonce that you generated previously.
  bytes = aead.Open(ciphertext, nonce);

  // If the message was originally string,
  // convert if from byte array to string.
  plaintext = Encoding.UTF8.GetString(bytes);

  // Print the decrypted message to the standard output.
  Console.WriteLine(plaintext);
}
```

As you can see, the API is incredibly simple to use. What's more,
even if you use the same nonce twice with the same key (or forget
to use it at all), the scheme doesn't fall apart; you will lose
[indistinguishability] if you repeat the same key/nonce/plaintext
combination, but your adversaries still won't be able to decrypt
the ciphertext.

The source code for the library is available on [GitHub]. For more
detailed usage examples see the [Miscreant.Examples] folder. API
documentation is available on the [C# Documentation] wiki page,
and the downloadable package can be found on
[Nuget](https://www.nuget.org/packages/Miscreant/).

[indistinguishability]: https://en.wikipedia.org/wiki/Ciphertext_indistinguishability
[GitHub]: https://github.com/miscreant/miscreant
[Miscreant.Examples]: https://github.com/miscreant/miscreant/tree/master/dotnet/Miscreant.Examples
[C# Documentation]: https://github.com/miscreant/miscreant/wiki/C%23-Documentation

![](/assets/img/2018-03-28-noise.png)

Another important use case for cryptography is client-server application
security. TLS is probably still the best solution for that problem, but
it's far from the ideal one. It can be used securely, provided that it's
correctly configured. That means you have to disable old versions, allow
only the safe cryptographic primitives, and probably [avoid the CA system]
entirely. Sometimes the TLS might not even be an option, and you will have
to develop a custom secure protocol from scratch. In such scenarios, the
best option by far is to use the [Noise Protocol Framework].

Noise is a framework for building modern cryptographic protocols
based on Diffie-Hellman key agreement. It's carefully designed,
lightweight, customizable, and easy to understand. Its early
adopters are [WhatsApp] and [WireGuard], among others. Noise has
open source implementations in many programming languages, but
until now, C# was not one of them.

[Noise.NET] is my .NET Standard 2.0 implementation of the Noise
Protocol Framework. It's a cross-platform, [libsodium]-based
library featuring:

- AESGCM and ChaChaPoly ciphers
- Curve25519 Diffie-Hellman function
- SHA256, SHA512, BLAKE2s, and BLAKE2b hash functions
- Support for pre-shared symmetric keys
- All known [one-way] and [interactive] patterns from the specification

It's very easy to use if you know what you are doing, which means that
you need to have at least the basic familiarity with the [specification],
because the library was designed to follow it as closely as possible
(if you prefer watching videos instead, Trevor Perrin's [RWC 2018 talk]
and David Wong's [brief tutorial] are both great introductions to the
framework). Here's one small example of the [Noise NK] handshake pattern
in action:

[avoid the CA system]: https://moxie.org/blog/authenticity-is-broken-in-ssl-but-your-app-ha/
[Noise Protocol Framework]: https://noiseprotocol.org/
[WhatsApp]: https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf
[WireGuard]: https://www.wireguard.io/papers/wireguard.pdf
[Noise.NET]: https://github.com/Metalnem/noise
[libsodium]: https://download.libsodium.org/doc/
[one-way]: https://noiseprotocol.org/noise.html#one-way-patterns
[interactive]: https://noiseprotocol.org/noise.html#interactive-patterns
[specification]: https://noiseprotocol.org/noise.html
[RWC 2018 talk]: https://www.youtube.com/watch?v=3gipxdJ22iM
[brief tutorial]: https://www.youtube.com/watch?v=ceGTgqypwnQ
[Noise NK]: https://www.discocrypto.com/#/protocol/Noise_NK

```csharp
// Choose the handshake pattern and cryptographic functions.
var protocol = new Protocol(
  HandshakePattern.NK,
  CipherFunction.AesGcm,
  HashFunction.Blake2s
);

// Start the handshake by instantiating the
// protocol with the necessary parameters.
var state = protocol.Create(initiator: true, rs: rs);

// Send the first handshake message.
var (bytesWritten, _, _) = handshakeState.WriteMessage(null, buffer);

// Receive the second handshake message.
var (_, _, transport) = handshakeState.ReadMessage(message, buffer);

// Send the transport message to the server.
bytesWritten = transport.WriteMessage(request, buffer);

// Receive the transport message from the server.
var bytesRead = transport.ReadMessage(response, buffer);
```

You can find the complete, runnable example in the [Noise.Examples]
folder of the [project's repository] on GitHub. The downloadable
package of the library is available on [Nuget]. Noise.NET has not
yet been reviewed, so code reviews and API feedback are welcome!

[project's repository]: https://github.com/Metalnem/noise
[Noise.Examples]: https://github.com/Metalnem/noise/tree/master/Noise.Examples
[Nuget]: https://www.nuget.org/packages/Noise.NET/
