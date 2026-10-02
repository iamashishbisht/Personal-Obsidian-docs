Checksums are compact values used to detect whether data changed. A match means no change was detected. It does not prove that change was impossible.

The kind of checksum matters. CRCs are strong tools for accidental corruption but not deliberate tampering. Cryptographic hashes such as SHA-256 are better for content identity and file verification. HMACs and digital signatures add proof that the data came from someone trusted.

Detection and recovery are separate jobs. Checksums tell you corruption happened. Replicas, retransmission, repair fragments, and backups are what recover the data.

Data does not always arrive exactly the way it was sent. A network packet can get damaged, a disk can return an old block, or a download can stop halfway through. The tricky part is that the bytes usually do not announce, "I am broken."

A **checksum** helps catch this. It is a small value calculated from a larger piece of data. Later, another part of the system calculates the value again and compares the two results.

Checksums show up everywhere in real systems: network frames, TCP and UDP packets, storage blocks, database pages, backups, container images, and replication logs.

> [!IMPORTANT] The important design question is: what kind of problem are you trying to catch? A CRC is good for accidental damage. A cryptographic hash is better when you need a strong content fingerprint. An HMAC or digital signature is used when you also need to know who produced the data.

> [!Question] 1. Tell me about CRC used for checksum 

Absolutely. **CRC is much closer to what you were thinking about for packet-level checking** than SHA-256 is.

## 1. What is CRC?

**CRC = Cyclic Redundancy Check.**

It is a relatively small value calculated from some data and attached to that data.

For example:

```
Sender

Data
  ↓
CRC calculation
  ↓
CRC = 10110110
  ↓
┌──────────────────────┐
│ Data | CRC           │
└──────────────────────┘
          ↓
       Network
```

The receiver performs the same CRC calculation on the received data:

```
Received Data
      ↓
CRC calculation
      ↓
Calculated CRC
      ↓
Compare with received CRC
```

If they don't match:

```
Calculated CRC ≠ Received CRC
             ↓
       DATA CORRUPTED
```

---

# 2. Why do we need CRC if we have SHA-256?

Because they solve **different problems**.

Think:

||CRC|SHA-256|
|---|---|---|
|Main purpose|Detect transmission/storage errors|Strong content fingerprint|
|Speed|Very fast|More computationally expensive|
|Output|Usually small|256 bits|
|Designed for accidental errors|✅|✅|
|Designed against intentional manipulation|❌|✅|
|Common at packet/link level|✅|Not typically|
|Common for file verification|Sometimes|✅|

CRC is basically:

> **"Did these bits probably get corrupted while being transmitted?"**

SHA-256 is more like:

> **"Is this entire content the same content I expected?"**

---

# 3. Your packet example

Let's say TCP sends a segment:

```
TCP Segment

┌──────────────────────────────────┐
│ Header                           │
│ Sequence Number = 5000           │
│ Checksum = ABCD                  │
│                                  │
│ Data = "Hello, how are you?"     │
└──────────────────────────────────┘
```

Imagine some bits get corrupted:

```
Sender:

Hello, how are you?
       ↓
Network
       ↓
Hello, how are yoX?
```

The receiver calculates the checksum over what it received.

If:

```
Received checksum ≠ calculated checksum
```

the receiver knows:

```
Something went wrong.
```

Then the **transport protocol's reliability mechanism** can cause the missing/corrupt data to be retransmitted.

So your original mental model was directionally correct:

```
packet
  ↓
integrity check
  ↓
corrupted?
  ↓
transport protocol handles recovery
```

But don't think:

> "CRC itself retries the packet."

CRC **only detects** the error.

The protocol decides what to do about it.

---

# 4. CRC is particularly good at detecting certain transmission errors

Suppose you transmit:

```
101100101101
```

During transmission:

```
101100101101
       ↑
     error
```

becomes:

```
101100111101
```

CRC is designed so that these kinds of accidental bit errors are detected with very high probability.

It is particularly effective at detecting things like:

- single-bit errors
- multiple-bit errors
- burst errors

That's why CRCs have historically been extremely common in communication and storage systems.

---

# 5. But CRC is NOT cryptographically secure

This distinction is extremely important.

Suppose:

```
Original data
     ↓
CRC
     ↓
1234
```

An attacker changes the data.

Because CRC is predictable, the attacker can potentially calculate a new CRC corresponding to the modified data.

So:

```
Original:

DATA → CRC 1234


Attacker modifies:

DATA' → CRC 5678
```

They can potentially modify **both the data and CRC**.

Therefore CRC answers:

> "Was this data accidentally corrupted?"

It does **not** reliably answer:

> "Did someone maliciously modify this data?"

For that, you'd use cryptographic mechanisms such as a MAC or digital signature, depending on the problem.

---

# 6. Now connect CRC, TCP and SHA-256

This is the mental model I want you to keep:

```
                    APPLICATION
                         │
                    File / Data
                         │
                    SHA-256
                         │
              "Is the final content
                 what I expected?"
                         │
                         ▼
                 ───────────────
                     TCP
                 ───────────────
                  Segment 1
                  checksum
                  sequence #

                  Segment 2
                  checksum
                  sequence #

                  Segment 3
                  checksum
                  sequence #
                         │
                         ▼
                      NETWORK
```

There are **different integrity checks at different layers**, serving different purposes.

And one subtle correction to your earlier statement:

**TCP's field is called a checksum, but it is not CRC.** TCP uses a one's-complement checksum, whereas CRCs are commonly used in other networking/link-layer and storage contexts.

So don't mentally equate:

```
checksum = CRC
```

Instead:

```
Checksum
 ├── TCP checksum
 ├── Internet checksum
 └── CRC
       └── one particular family of error-detecting checks
```

That's the cleaner mental model.


> [!Question] 2. Cryptographic hashes such as SHA-256 are better for content identity and file verification in checksum how?

The key idea is: **a cryptographic hash gives you a compact fingerprint of content**.

Suppose you have a file:

```
report.pdf
```

You calculate:

```
SHA-256(file bytes)
        ↓
A3F7...9C21     ← 256-bit hash
```

If even **one bit** of the file changes, the SHA-256 output should change dramatically.

### Why is that useful for checksums?

Imagine I send you a 2 GB file.

Instead of comparing all 2 GB byte-by-byte, I can give you:

```
SHA-256 = ABC123...
```

You download the file and calculate:

```
SHA-256(downloaded_file)
```

Then:

```
ABC123... == ABC123...
       ↓
very strong evidence that
the contents are identical
```

If instead:

```
ABC123... != XYZ789...
```

then the file contents are different.

---

### Why specifically a cryptographic hash?

A normal checksum can also detect accidental corruption, but cryptographic hashes provide much stronger properties.

For SHA-256:

**1. Deterministic**

Same content → same hash.

```
"hello" → 2cf24d...
"hello" → 2cf24d...
```

**2. Avalanche effect**

Tiny input change → drastically different output.

```
"hello"
    ↓
2cf24d...

"Hello"
    ↓
185f8d...
```

**3. Extremely difficult to deliberately find another input with the same hash**

This is important when you aren't merely checking for accidental corruption but want stronger assurance against intentional manipulation.

**4. Fixed size**

Whether your file is:

```
10 KB
100 MB
10 GB
```

SHA-256 always produces:

```
256 bits = 32 bytes
```

or **64 hexadecimal characters**.

---

### Content identity

This is another very useful concept.

Instead of identifying a file by:

```
filename = report.pdf
```

you can identify its **contents** by:

```
SHA-256(content) = 8f14e45f...
```

For example:

```
File A
"Java is great"
       ↓
SHA-256
       ↓
HASH-X


File B
"Java is great"
       ↓
SHA-256
       ↓
HASH-X
```

So you know:

```
A and B have the same content
```

This is why content-addressed systems can use hashes as identifiers.

---

### But one important correction

Don't interpret:

> "Same SHA-256 = mathematically guaranteed same file."

That's not literally true.

SHA-256 maps an enormous input space into only `2²⁵⁶` possible outputs, so **collisions are theoretically possible**.

The practical statement is:

> **For ordinary file verification, matching SHA-256 hashes provide extremely strong evidence that the files have identical contents.**

And that's why you'll see SHA-256 used for things like **software downloads, artifacts, container images, Git-related content addressing, and data integrity verification**.

If you want, I can also explain **why SHA-256 is better than MD5/SHA-1 for checksums using an actual file-download example**, which makes the distinction very clear.


> [!Question] 
> 3. ok but instead of having sha 256 in entire payload, it is added in each packet right, as if it is sent in packet by sender then that packet can be retried again if it doesn't match.

Not quite — you're mixing **file-level hashing** with **packet-level integrity checks**.

### 1. SHA-256 is normally calculated for the entire content

Suppose you send a 100 MB file:

```
100 MB file
     ↓
SHA-256
     ↓
64-character hash
```

The receiver downloads/reconstructs the file and calculates SHA-256 again:

```
Received 100 MB
     ↓
SHA-256
     ↓
compare with sender's hash
```

If they differ, you know **somewhere the content was altered/corrupted**, but SHA-256 itself doesn't tell you which packet was bad.

---

### 2. Packets have their own integrity mechanisms

At the network/transport level, packets/segments already have mechanisms for detecting corruption.

For example, TCP has a **checksum** on each TCP segment:

```
TCP segment
┌─────────────────────────┐
│ Header                  │
│ Sequence number         │
│ Checksum                │
│ Data                    │
└─────────────────────────┘
             ↓
        Network
             ↓
┌─────────────────────────┐
│ Receiver                │
└─────────────────────────┘
```

The receiver can detect that a TCP segment was corrupted.

But importantly:

> **The receiver doesn't normally say "this packet's SHA-256 doesn't match, so resend this packet."**

TCP uses **sequence numbers + acknowledgements + retransmission mechanisms** to recover missing/lost data.

For example:

```
Sender                         Receiver

Segment 1  ─────────────────→  received
Segment 2  ───────X             lost
Segment 3  ─────────────────→  received

                               "I need segment 2"

Segment 2  ─────────────────→  retransmitted
```

The application ultimately gets an ordered byte stream.

---

### 3. So there are two different layers

Think about it this way:

```
              YOUR FILE
                  │
                  │ SHA-256
                  ↓
        "Is the final content
         what I expected?"
                  │
                  ↓
        ───────────────────
              TCP
        ───────────────────
          Segment 1
          checksum
          sequence number

          Segment 2
          checksum
          sequence number

          Segment 3
          checksum
          sequence number
                  │
                  ↓
              Network
```

So:

**TCP checksum** → helps detect corruption at the transport-segment level.

**TCP sequence numbers + ACK/retransmission** → help recover missing/lost data.

**SHA-256 of the complete payload/file** → gives you a strong end-to-end fingerprint of the final content.

---

### And this connects directly to what we discussed about HTTP/2 and QUIC

This distinction is important:

```
HTTP
 ↓
TCP
 ↓
IP
 ↓
Network
```

TCP handles reliable delivery of the byte stream.

With **QUIC**:

```
HTTP/3
  ↓
QUIC
  ↓
UDP
```

QUIC implements its own reliability/integrity mechanisms because UDP itself doesn't provide TCP-like reliability.

So **SHA-256 isn't normally the mechanism used to make individual HTTP packets retransmittable**. That's a different job from content hashing.