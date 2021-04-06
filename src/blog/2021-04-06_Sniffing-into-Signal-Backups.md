# Sniffing into Signal Backups

For quite some time[^0] now the [Signal](https://signal.org/) Android App offers encrypted backups. This is a
comprehensive backup, which includes amongst others all message history, attachments, and most notably the app's key
pair. Because of its focus much on UX in the app, and how it embeds into the Android ecosystem (reliance on Google Cloud
Messaging, availability only in the Google Play Store, etc.), Signal experienced tremendous adaption in contrast to its
competitors like [Element](https://element.io/) -- Signal's premise is Security, not Privacy[^1]. The recent adoption
waves, mostly triggered by WhatsApp's ToS changes show, that users seemingly don't care about the technical details of
the messager used -- WhatsApp also uses Signal's [Double Ratchet
Algorithm](https://signal.org/docs/specifications/doubleratchet/) to encrypt messages. The key differences between
WhatsApp and Signal are: Signal's code base is open source (you have to trust them to run the
[server](https://github.com/signalapp/Signal-Server) code like they claim), and backing entity is a Nonprofit. Don't get
me wrong, those are very relevant differences, but nothing ground-breaking in terms of privacy.

Anyway, this post was supposed to be about how the backup of the Signal Android App is structured[^2]. We're going to
decompose the backup file by writing a parser for it in Rust, layer by layer:

The backup file itself is a binary blob, which is a sequence of Protobuf encoded messages:
```proto
message BackupFrame {
    optional Header           header     = 1;
    optional SqlStatement     statement  = 2;
    optional SharedPreference preference = 3;
    optional Attachment       attachment = 4;
    optional DatabaseVersion  version    = 5;
    optional bool             end        = 6;
    optional Avatar           avatar     = 7;
    optional Sticker          sticker    = 8;
    optional KeyValue         keyValue   = 9;
}
```
(This is the widely adopted protobuf way of encoding a poor man's enum ..)

The general structure of the file looks like:
```no_run,noplayground,ignore
|4:len|len:header_bytes|        # Header
--
|4:len|len:frame_bytes |10:mac| # n consecutive encrypted BackupFrames
```

## The Header
The first frame is not encrypted, as it provides both a salt and an initialization vector for the crypto scheme, which
is [AES in counter
mode](https://github.com/signalapp/Signal-Android/blob/d74e9f74103ad76eb7b5378e06fb789e7b365767/app/src/main/java/org/thoughtcrime/securesms/backup/FullBackupExporter.java#L448)
(aes-ctr); but let's not get ahead of ourselves and get the initial frame:
```rust,no_run,noplayground,ignore
let header_len = reader.read_u32::<BigEndian>()? as usize;
let mut header_frame = vec![0u8; header_len];
reader.read_exact(&mut header_frame)?;

let frame = model::BackupFrame::decode(&header_frame[..])?;
```
Cool, now we have the header frame, now let's extract the used salt and the initialization vector:
```rust,no_run,noplayground,ignore
// Hash generation as input for key derivation
let mut hasher = Sha512::new();
let pass_without_whitespace = key.replace(" ", "");

let salt = frame
    .header
    .as_ref()
    .and_then(|h| h.salt.as_ref())
    .context("Invalid header")?;
hasher.update(salt);

let mut hash = pass_without_whitespace.as_bytes().to_vec();
for _ in 0..250000 {
    hasher.update(hash);
    hasher.update(&pass_without_whitespace);
    hash = hasher.finalize_reset().to_vec();
}
// Take the first 32 bytes as input for key derivation
let key = &hash[..32];
let derived = derive_secrets(key)?;
let mac_key: [u8; 32] = derived[32..].try_into().unwrap();
let cipher_key: [u8; 32] = derived[..32].try_into().unwrap();

let init_vector: [u8; 16] = {
    let iv = frame.header.and_then(|h| h.iv).context("Invalid header")?;
    anyhow::ensure!(iv.len() == 16);
    iv[..].try_into().unwrap()
};
```
After hashing the user provided password (with an initial salt) 250000 times, said hash is used as input for the key
derivation function. The resulting key is split, the first 32 bytes used as input for a hash based message
authentication code (HMAC), the second 32 bytes as input for the encryption scheme. Lastly, we extract the
initialization vector, which is used as a starting nonce later.
The key is derived by using HMAC-based Extract-and-Expand Key Derivation Function (HKDF)[^3] with a nulled salt, and
static info bytes:
```rust,no_run,noplayground,ignore
fn derive_secrets(key: &[u8]) -> anyhow::Result<[u8; 64]> {
    let salt = [0u8; 32];
    let h = hkdf::Hkdf::<Sha256>::new(Some(&salt), key);
    let mut okm = [0u8; 64];
    h.expand(b"Backup Export", &mut okm)
        .map_err(|e| anyhow::anyhow!(e))?;
    Ok(okm)
}
```

No we have everything place in order to start parsing the rest of the file.

## Individual `BackupFrame`s

For each frame until EOF, we start off again with extracting the length of the blob, and then read the bytes into
memory:
```rust,no_run,noplayground,ignore
let frame_len = self.reader.read_u32::<BigEndian>()? as usize;
let mut frame = vec![0u8; frame_len];
self.reader.read_exact(&mut frame)?;
```
Next, the blob's authenticity is checked, whereas only the first 10 bytes of the derived hash are persisted in the
file[^4]:
```rust,no_run,noplayground,ignore
let their_mac = &frame[frame.len() - 10..];
self.hmac.update(&frame[..frame.len() - 10]);
let our_mac = self.hmac.finalize_reset().into_bytes();

// only first 10 bytes of the mac are persisted in the file
anyhow::ensure!(our_mac.starts_with(&their_mac), "Bad MAC");
```
The encryption scheme is AES in [counter
mode](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_(CTR)), where a monotonically increasing
counter is combined with the initialization vector derived from the file's header frame serving as input nonce to
decrypt an encrypted block. With each new frame, the counter is increased:
```rust,no_run,noplayground,ignore
let iv = {
    let mut tmp = [0u8; 16];
    // First 4 bytes are used as counter
    BigEndian::write_u32(&mut tmp[..4], self.counter);
    // Filled with IV from 4 to 16
    tmp[4..].copy_from_slice(&self.init_vector[4..]);
    self.counter += 1;
    tmp
};

let key = GenericArray::from_slice(&self.cipher_key[..]);
let nonce = GenericArray::from_slice(&iv[..]);
let mut cipher = Aes256Ctr::new(&key, &nonce);
```
And with this `cipher`, we can decrypt the frame's data bytes inplace and parse the protobuf encoded message:
```rust,no_run,noplayground,ignore

let read_to = frame.len() - 10;
cipher.apply_keystream(&mut frame[..read_to]);

let frame = model::BackupFrame::decode(&frame[..read_to])?;
```

With that, we have the basic building blocks to extract information from Signal's encrypted backups, like dumping the
messages into a database, exporting all attachments, etc..

The full code of this can be found in [this repo](https://github.com/wngr/signal-beagle).

-----

[^0]: The initial implementation in [this
  commit](https://github.com/signalapp/Signal-Android/commit/24e573e537639f6f8ff40fd774cf9ff079bbacce).

[^1]: Over the years, Moxie outlined his stance on this several times. Examples include [this
  comment](https://github.com/signalapp/Signal-Android/issues/127#issuecomment-13335689) and [this other
  comment](https://github.com/signalapp/Signal-Android/issues/281#issuecomment-21762482).

[^2]: All observations are based on
  [`d74e9f74103ad76eb7b5378e06fb789e7b365767`](https://github.com/signalapp/Signal-Android/tree/d74e9f74103ad76eb7b5378e06fb789e7b365767).

[^3]: [https://tools.ietf.org/html/rfc5869](https://tools.ietf.org/html/rfc5869)


[^4]: I suppose this was done to save some bytes, but this opens the door for potential collision attacks.
