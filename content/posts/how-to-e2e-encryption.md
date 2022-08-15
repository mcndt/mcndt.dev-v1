---
title: "Designing an end-to-end encrypted note sharing service 🔐📝"
date: '2022-08-14'
draft: true

summary: "TODO"

tags: ["Noteshare.space", "Information Security"]
---

I have used [Obsidian](https://obsidian.md/) for my personal knowledge management (PKM) for over a year now. I love it because of it's simplicity (everything is stored on disk as plain Markdown files), the way it promotes connecting my knowledge, and its local-first mentality. 

However, that last part is also a big pain point for me. There is no trivial way for me to share my knowledge apart from sending the plain Markdown file; my knowledge base turns into its own walled garden. For example, I take most of my study notes in Obsidian, which works great for me, but is annoying for my classmates when I have to send them plain `.md` files of my notes. 

Other Obsidian users have felt this pain too. For example, Tim Rogers created the [Share as Gist](https://github.com/timrogers/obsidian-share-as-gist) plugin, and a plugin to [share to Notion](https://github.com/Easychris/obsidian-to-notion) has been created as well. However, none of these solutions are very user-friendly, because they require accounts with third-party services, fiddling with API tokens, and offer no end-to-end encryption. 

That's why I decided to build [Noteshare.space](https://noteshare.space), my own, end-to-end secured implementation of a "share a note to the web" service. The implementation was inspired by [Thomas Konrad's talk](https://youtu.be/mffWMMVMMLs) about how the (now retired) Firefox Send file sharing service worked.

## Functional requirements

Now that the stage has been set, let's review the formal requirements for this project:

- **One-click sharing** from the Obsidian UI (as easy as sharing a google doc via URL)
- **End-to-end encryption** (so you can safely share e.g. work documentation, or a love letter)
- **Temporary storage** (Noteshare is not intended as a syncing, backup, or permanent hosting solution)
- Shared notes can be opened by non-Obsidian users (i.e. we will build a **web viewer**)

As we will see, it is actually quite straight-forward to meet all of these. Now, let's take a look at the architecture I came up with.

## Architecture

The architecture I came up with consists of four components: (i) an Obsidian plugin for encrypting the notes, (ii) a note storage service, (iii) a SvelteKit web application for rendering the web view, and (iv) the web browser for opening the shared notes and decrypting them.

{{<figure width=720 align=center src="/posts/media/noteshare-architecture.png" title="Software architecture for Noteshare.space" caption="" attrlink="" attr="Copyright © 2022 mcndt">}}

The flow for storing and retrieving notes is roughly as follows:

1. The user wants to create a share link in the Obsidian application. The plugin **encrypts** the note with a **single-use key** (see later), and send the encrypted note to the storage service. The storage service then returns the URL at which the note can be retrieved.
2. The user **shares the note URL** and the decryption key needed to recover the plain text Markdown.
3. On opening the URL in a web browser, the web application **retrieves the ciphertext** and embeds it in a web page.
4. The web browser **decrypts** the ciphertext with the decryption key and **renders** the Markdown to HTML.

Pretty straight-forward, right? Next, let's take a look at how we can implement a secure encryption mechanism for this architecture.

## Security mechanisms

Information security is notoriously hard to get right, which is why you should always stick to well-researched encryption algorithms and security schemes. For this project, I used standard **AES [symmetric encryption](https://en.wikipedia.org/wiki/Symmetric-key_algorithm)** with a **[keyed message authentication code (MAC)](https://en.wikipedia.org/wiki/HMAC)** to validate the authenticity of the ciphertext.

### AES-CBC encryption

Let's take a look at the encryption first. The following snippet shows the encryption steps as pseudocode:

```ts
function encrypt(plaintext, key) {
  // Encrypt the message in CBC mode
  const ciphertext = AES(plaintext, key, {mode: "CBC"});
  // Compute keyed message digest
  const hmac = HMAC_SHA_256(ciphertext, key)
  
  return ciphertext, hmac;
}
```

In a first step, we encrypt the plaintext Markdown with the AES algorithm and a 256-bit key. We run the algorithm in conjunction with **[cipher block chaining (CBC)](https://www.techtarget.com/searchsecurity/definition/cipher-block-chaining)** to encrypt files of arbitrary length. A downside of CBC  is that the encryption cannot be parallelized. Other encryption schemes, such as [Galois/Counter mode (GCM)](https://csrc.nist.rip/groups/ST/toolkit/BCM/documents/proposedmodes/gcm/gcm-spec.pdf) are much faster. However, they usually come with subtle pitfalls; when used incorrectly, it becomes trivial for attackers to decrypt the ciphertext. Regardlees, because our Markdown file is in the order of kilobytes, using CBC is not an issue.

### Keyed message digest

In the second step, we compute a [keyed hash (HMAC)](https://en.wikipedia.org/wiki/HMAC) of the obtained ciphertext. 
This way, even if an attacker modifies the ciphertext, it is impossible to compute a new corresponding HMAC without knowing the symmetric encryption key. 
Using the encryption key, the recipient of the encrypted note (i.e. the web browser) can verify that the ciphertext was not tampered with before they attempt to decrypt it. 
The user can now safely send the ciphertext and HMAC values to the storage server without the server being able to read the content of the Markdown.

### Generating single-use keys

There remains just one small problem: what should we use as the symmetric key? It is absolutely [not safe](https://en.wikipedia.org/wiki/Random_number_generator_attack#:~:text=Modern%20cryptographic%20protocols%20often%20require,as%20random%20number%20generator%20attacks.) to use `Math.random()` for this. Most standard random number generators are only [pseudorandom](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) and can [easily be exploited](https://www.synopsys.com/blogs/software-security/pseudorandom-number-generation/) to derive the secret key used by the client.

To generate a secure key, we need a cryptographically secure random number generator. In my implementation, I used the common **[PBKDF2 algorithm](https://cryptobook.nakov.com/mac-and-key-derivation/pbkdf2)** using the Markdown content as the seed. This is secure as long as the attacker does not know the Markdown content, in which case there is no point to cracking the key in the first place. To ensure the key is unique even when the same note is shared twice, I concatenate the current timestamp to the seed.

```ts
function generateKey(seed) {
  // Note: No salt is added because the generated key is for one-time use anyways.
  return PBKDF2(seed, salt="", { key_bits: 256 });
}

// Putting it all together
const key_256 = generateKey(time.now() + plaintext)
const ciphertext, hmac = encrypt(plaintext, key_256)
```

### Alternative solution with public/private keys

A remaining issue is that the shared symmetric key can be reused by malicious recipients to change the ciphertext and compute a valid HMAC. This can then be injected into the traffic of other users without being detected during decryption.

As a solution, we can generate a public-private keypair (e.g. a 2048-bit RSA keypair), use the private key to obtain the ciphertext, and the public key to compute the HMAC. The private key is then thrown away after the note is stored on the server. Recipients can use the public key to verify the message digest and decrypt the ciphertext, but cannot generate a new ciphertext that can be decrypted by the public key.

While neat, this solves a problem that is beyond the goal for Noteshare, which is to securely exchange text. Therefore I used the symmetric encryption approach.

## Sharing encryption keys 

- How do we then share the key if the server is not allowed to be able to decrypt your note?
- For this we use a clever feature of web browsers you have most likely used before:
- e.g. see the following URL: `https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Syntax`
- Part of the URI syntax is the `["#" fragment]` suffix.
- It is most commonly used by web browsers to automatically scroll to a specific header on a webpage, in this case the “Syntax” subsection of the Wikipedia article.
- What is unique about the fragment component of the URI is that web browsers never include it in their HTTP requests.
- In other words, we can add data to URLs that are accessible to the browser, but not the server.
	- –> Perfect for our encryption key!
- In this way, we can easily share the note with the decryption key: `https://noteshare.space/note/[NOTE_ID]#[DECRYPTION_KEY]`
- The returned web page includes the encrypted note in base64. The included JavaScript then pulls the decryption key from the URL fragment and retrieves the plaintext note.

## Noteshare.space

- It is on this foundation that I built Noteshare.space. I initially built it for my own personal use, but have since opened use to the entire Obsidian community.

## Conclusion

- In this article, I established the groundwork of how I built an end-to-end encrypted data storage service.
- However, note that this is far from everything you need to build an encrypted data hosting service!
	- You need to protect your service from abuse, both in terms of volume of data (rate/volume limiting) and volume of requests (DOS protection).
	- Furthermore, you might want to implement some mechanism to block known bad actors from your system.
