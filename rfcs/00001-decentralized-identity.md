---
title: Decentralized Identity
status: draft
parent-rfc: "none"
---

# Summary
[summary]: #summary

A protocol for a decentralized (but not necessarily distributed)
public-key system that is (mostly) independent of centralized
authorities.  It provides authentication for identities (usernames)
based off of a location, using the authority of that location.
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

Need to pin down "client" and "server" in better language.

 * Identity
 * Identity server

Figure out identity format better.  Right now I'm pretending that it's
something that looks like email addresses, but email addresses are way
more complicated than they look and this must be as foolproof as
possible.


# Detailed design
[design]: #detailed-design

## Basic design

You have an identity server that stores a mapping of identity's to
public keys.  You have a protocol to allow anyone to ask the server
for the public key for a particular identity.  That is basically it.

This says NOTHING about how private keys are managed; that is entirely
between the identity server and the owner of the private key.  Maybe
it's in a file on the user's hard drive.  Maybe it's in a hardware
security device like a Yubikey.  Maybe it's stored by the the identity
server and the user has to log in with a password to retrieve it.  The
point is that *nothing using this system needs to care* how the
private keys are managed, so we leave it undefined.

As an EXAMPLE, a user `joe@example.com` may want to submit a new post
to a web forum.  He logs in to his identity server with a password and
gets his private key.  His "submit new post" API call gets signed with
the private key and sent to `different-example.com`.  The forum
software phones up `example.com`'s identity server and asks for the
public key for `joe@example.com`.  It uses the public key to verify
that the message is valid, and if so, accepts it.

## Communication protocol

JSON-RPC over HTTP(S).

Defining a specific "envelope" format for a signed message may also be
useful.

The actual public keys themselves may get stored in IPFS.  There's
nothing particularly wrong with that.  ...In fact, an IPFS hash
for a particular public key might be a *very* good unique
identifier that makes this whole system unnecessary.  Downside, each
key becomes basically a unique identity...  BUT, if it is signed by a
previous key, then it becomes possible to make a continuous chain of
trust for a given identity.

Example, you start with a key for an identity:

```json
{
    "id": "joe@example.com",
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
    "id": "joe@example.com",
    "created": "2017-12-04T00:00:01Z",
	"key": "...",
	"previous": "QmKey1",
	"signature": "this object signed by QmKey1"
}
```

This gets published as an IPFS object with the hash `QmKey2`.  

Now when the user submits a request they sign it with the new key tell
the server to use the key `QmKey2`.  The server checks the public key
and says "oh this is not something I recognize from `joe@example.com`,
but it has been signed by the previous key so I know it's legit".

I actually like that quite a bit...

It has some downsides though: The server can't know if a key has been
replaced unless it is expired.  Fragile!  The key becomes the identity
and the `id` field is mostly meaningless; two people may claim to have
the same identity and have entirely separate keys.  How much this
matters is debatable; maybe you can have Discord-style usernames such
as `joe#QmKey1`, and whenever `joe` submits a message, even if it's
signed with `QmKey2`, the device wanting to authenticate it knows that
if it follows the signature chain backwards it should always end up at
`QmKey1`.  If a different `joe` makes `joe#QmSomeOtherKey` then this
is an entirely different identity.  It also means that if an identity
gets lost there's no recovering it.

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

## Locating identity servers

DNS SRV records

## Updating keys

Part of the problem with GPG is that key management is a pain in the
ass.  But, being able to expire and replace keys is critical for
defensive security (replace a key if it gets compromised), pre-emptive
hardening (it's harder to compromise a key if they change every N
months), and usability (a hard drive crash destroys a key, it needs to
be replaced with a new one).

Keypairs should have creation dates and optionally expiration dates.
When a key is updated it should be signed by the previous key (perhaps
that should be a MUST?) to keep the chain of authenticity intact.  An
identity server should always return the most recent key, but it
should be possible to ask for older ones.

Alterations to the public key on the server may be performed via
messages signed with the matching private key.  Defining a specific
API for this may be useful.

# Drawbacks
[drawbacks]: #drawbacks

It's not perfect.  See "security considerations" and "unresolved
questions" for the most important issues.

I'm not sure this really jibes well with the existing Web security
model.  Namely, if joe@example.com logs in to his identity server with
a password and his private key is stored in, say, session local
browser storage, then there's no way for anything that isn't from
example.com to access it to sign messages with it.  This is as it
should be; Javascript from random sites shouldn't be able to read your
private key!  But you need to have some way to allow messages to be
signed with this key.  There's probably clever ways to work around
this if necessary.

# Security considerations
[security]: #security

Oh, there's tons of things here I'm sure, which I do *not* know enough
about.

The *main* security consideration is that the "source of trust" in
this system comes *completely* from the domain name system and whoever
runs the identity server for a given domain.  If you can spoof DNS,
you win.  If you can compromise an identity server, you win.  If you
can subvert the authority of the DNS system (ie ICANN gets told by the
police to alter domain records), you win.  Identity servers are a
great single point of failure.

However, this is no worse than existing systems.

This seems to be entirely compatible with systems like
[JSON Web Tokens](https://en.wikipedia.org/wiki/JSON_Web_Token) or
such.  This just handles the "log in using credentials" portion of the
system where a user obtains the token in the first place.  These
systems should work well together; after authentication, the server no
longer needs to remember a user's public key and the user no longer
needs to sign their messages, just provide the token.  (These together
honestly start looking a bit like Kerberos, from what I understand.)

What kind of security questions does this raise?  Things to think about: Imitation/spoofing, network control (sybil attacks), denial of service attacks, information leakage (users being tricked into revealing things beyond the necessary.)

# Alternatives
[alternatives]: #alternatives

 * OAuth
 * OpenID (might be worth looking at?)
 * GPG -- find out more about API and automation level stuff.  My
   impression is that GPG basically does all of this and more (you
   have a web of trust instead of authorative servers), but in a very
   ad-hoc way that is very inconvenient for users to deal with and
   very inconvenient for programs to interact with.  (How DO you
   request a key from a GPG key server?  I can't find documentation on
   the protocol.)  Disadvantages are: Key management is hard, all the
   message protocols and file formats and such are very email-oriented
   and not really appropriate for signing API requests, it's old and
   complicated with a million little frobs and switches and options
   that are a huge pain in the butt, and despite being 30 years old
   nobody still bloody uses it so there's no real cost to replacing it
   with something new.  Despite all this, it might be best to use GPG
   anyway.
   

# Unresolved questions
[unresolved]: #unresolved-questions

**I find it incredible that nobody has ever thought of this before.
What systems don't I know about?**

Can we make it more distributed (ie, no privilieged servers)?

Can we make it rely less on the authority of the identity servers, for
instance by allowing keys to sign each other producing a web of trust?

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?

# References/related works
[references]: #references

What other things are related to this that might be worth looking at?
