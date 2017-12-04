---
title: Distributed Identity
status: draft
parent-rfc: "none"
---

# Summary
[summary]: #summary

A protocol for a distributed public-key distribution system that is
independent of centralized authorities.  It provides authentication
for identities (usernames) based off of a public key stored as an IPFS
object, and allows publishing new keys with a chain of trust.
Critical features are: any system, human or machine, can take a
message from a user signed with their private key, and retrieve that
user's public key to verify the message.

# Motivation
[motivation]: #motivation

Web identity is a headache.  It's ridiculous that the *easiest* way to
manage identity in any web service is to roll your own system, and yet
there's not many good pre-made systems for doing this.  OAuth claims
to solve this problem, but it's so complicated to implement and
operate that the only the largest sites (gmail, facebook, etc) really
provide it.  Furthermore, OAuth basically just farms out a request to
a different webpage, as far as I can tell, which makes its usage quite
silly for, say, a command line program.

For both interactive user applications (say, a web forum) and
non-interactive systems (a program calling another program's remote
API), the essential question that needs to be asked is basically "does
this message (HTTP request, usually) come from who it claims to come
from?"  With public key encryption, this does not seem like it should
be difficult to answer!  If a user has a private key, and there's an
easy way for a server to find and retrieve the user's public key, then
this becomes basically transparent.  There's no "login" for a
particular website, no passwords that get stored all over the place
with unknown levels of security, nothing like any of that.  All the
server needs to know is "do I care about the user attempting to make
this request?"  Authorization, in other words.  Authentication becomes
trivial.

# Terms and definitions
[terms]: #terms

 * Identity: A unique identifier that is tied to a particular user or
   set of permissions.
 * Key: Usually actually means a public/private key pair.
 * User identifier: Probably should be a better word for this.
   Regardless, it's a textual label for a particular identity, going
   with the tentative format of `<identifier>#<CID>` where
   `<identifier>` is a short, basically arbitrary human-readable text
   (possibly including nothing), and the `<CID>` is the CID (content
   identifier) of an object stored in IPFS.  The human-readable
   portion of this is probably not significant, but can be useful.

# Detailed design
[design]: #detailed-design

## Basic design

You have public keys along with metadata stored and distributed via
IPFS.  User identifiers contain the hash of the public key object.
Anyone can retrieve the public key.  That is basically it.

This says NOTHING about how private keys are managed; that is entirely
up to the owner of the private key.  Maybe it's in a file on the
user's hard drive.  Maybe it's in a hardware security device like a
Yubikey.  Maybe it's stored by the a server somewhere and the user has
to log in with a password to retrieve it.  The point is that *nothing
using this system needs to care* how the private keys are managed, so
we leave it undefined.

As an EXAMPLE, a user `joe#QmFoo` may want to submit a new post to a
web forum at `example.com`.  He plugs in his security token and
unlocks his private key.  His "submit new post" API call gets signed
with the private key and sent to `example.com`.  The forum software
retrieves the public key with the CID `QmFoo` from IPFS.  It uses the
public key to verify that the message indeeds come from `joe#QmFoo`,
and if so, accepts it.

## Key format

JSON.

Actually, JSON might be problematic because of insignificant
whitespace making two different formats of the same object maybe have
different hashes.  CBOR may be better.  Or, JSON normalized down to a
specific form, but at that point you might as well convert it to CBOR
or such anyway.

Algorithms?  Keep it bare-bones as possible.  Don't let an attacker
choose an algorithm whenever possible.  Don't even bother with
less-than-ideal algorithms.

Defining a specific "envelope" format for a signed message may also be
useful.

The actual public keys themselves get stored in IPFS.  Downside, each
key becomes basically a unique identity...  BUT, if it is signed by a
previous key, then it becomes possible to make a continuous chain of
trust for a given identity.

Example, you start with a key for an identity:

```json
{
    "id": "joe",
    "created": "2017-12-03T13:59:01Z",
	"key": "..."
}
```

This gets published as an IPFS object with hash `QmKey1`.  You have
the same system of a user submitting a request that's signed with the
private key, and the server retrieving the public key to verify it.

Now you want to update the public key.  You publish a new key and sign
it with the old one:

```json
{
    "id": "joe",
    "created": "2017-12-04T00:00:01Z",
	"key": "...",
	"previous": "QmKey1",
	"signature": "this object signed by the private key for QmKey1"
}
```

This gets published as an IPFS object with the hash `QmKey2`.
Now when the user submits a request they sign it with the new key.
The server checks the public key and says "oh this is not `joe#QmKey1`
but if I follow the chain backwards then it has been signed by
`QmKey1` so I know it's a legit continuation of the same identity".

## Updating keys

Key management is a pain in the ass.  But, being able to expire and
replace keys is critical for defensive security (replace a key if it
gets compromised), pre-emptive hardening (it's harder to compromise a
key if they change every N months), and usability (a hard drive crash
destroys a key, it needs to be replaced with a new one).

This system cannot address all these problems.  All it can do is allow
keys to have expiration dates, and allow a user to advertise a newer
key that is authenticated by an older one.  There is no way to revoke
compromised keys, no way to recover lost/destroyed private keys, and
no way to start with an old key and find newer ones based off of it.
These problems are intended to be dealt with by a future protocol.

# Drawbacks
[drawbacks]: #drawbacks

See `Updating keys` section above for the main ones.  To summarize:

 * You can't expire or revoke keys by any mechanism except an expiry
   date
 * You can't recover or update lost keys
 * The hashes aren't SUPER human readable, which is potentially
   problematic... if someone's identity is
   `joe@QmT78zSuBmuS4z925WZfrqQ1qHaJ56DQaTfyMUF7F8ff5o` then it may be
   possible to relatively easily forge an object with an IPFS hash
   `QmT78...ff5o` that is easy to visually mistake for the proper
   hash.
 
Most of these functions require some sort of updateable database or
other method of indirection, which IPFS cannot provide on its own.
This is intended to be the topic of a future RFC which will provide an
updateable way to associate a name (for example, one linked to a
domain name, such as `joe@example.com`) to an identity stored in IPFS.

The other downside with a fully-distributed system is that sometimes
having an identity backed by a particular domain/institution is
*useful*.  `joe@microsoft.com` or `joe@harvard.edu` says a lot more
than `joe#QmKey1` does.

# Security considerations
[security]: #security

Oh, there's tons of things here I'm sure, which I do *not* know enough
about.

This seems to be entirely compatible with systems like
[JSON Web Tokens](https://en.wikipedia.org/wiki/JSON_Web_Token) or
such.  This just handles the "log in using credentials" portion of the
system where a user obtains the token in the first place.  These
systems should work well together; after authentication, the server no
longer needs to remember a user's public key and the user no longer
needs to sign their messages, just provide the token.  (These together
honestly start looking a bit like Kerberos, from what I understand.)

# Alternatives
[alternatives]: #alternatives

 * OAuth
 * OpenID (might be worth looking at?)
 * GPG -- find out more about API and automation level stuff.  My
   impression is that GPG basically does all of this and more, but I
   have found it to be really inconvenient for users to deal with and
   very inconvenient for programs to interact with.  (How DO you
   request a key from a GPG key server?  I can't find documentation on
   the protocol.)  Still, it deserves a good hard look, since I don't
   know enough about it; there's lessons there to be learned.


# Unresolved questions
[unresolved]: #unresolved-questions

**I find it incredible that nobody has ever thought of this before.
What systems don't I know about?**

How do we nicely have an updateable database to associate nicer
usernames to hashes?

An actual web of trust would be nice!!!  How do we do that without
publishing new keys?  Or do we just publish new keys whenever someone
signs your key?  That might work out well actually...

# Future work
[future]: #future-work

As mentioned, mapping updateable, location-based usernames such as
`joe@example.com` to a particular hash would have some benefits.

# References/related works
[references]: #references

What other things are related to this that might be worth looking at?
