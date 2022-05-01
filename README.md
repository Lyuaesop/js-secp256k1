# JS-secp256k1 !

[Fastest](#speed) JS implementation of [secp256k1](https://www.secg.org/sec2-v2.pdf),
an elliptic curve that could be used for asymmetric encryption,
ECDH key agreement protocol and signature schemes. Supports deterministic **ECDSA** from RFC6979 and **Schnorr** signatures from [BIP0340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki).

## Usage
```js
<script src='[your path]/S256K1.js'></script>
<script>
(async () => {
  const privateKey = S256K1.utils.randomPrivateKey();
  const messageHash = await S256K1.utils.sha256("hello world");
  const publicKey = S256K1.getPublicKey(privateKey);
  const signature = await S256K1.sign(messageHash, privateKey);
  const isValid = S256K1.verify(signature, messageHash, publicKey);

  const signatureE = await S256K1.sign(messageHash, privateKey, { extraEntropy: true });
  const signatureM = await S256K1.sign(messageHash, privateKey, { canonical: false });
  const signatureM = await S256K1.sign(messageHash, privateKey, { der: false });

  console.log(S256K1.utils.bytesToHex(publicKey));

  const rpub = S256K1.schnorr.getPublicKey(privateKey);
  const rsignature = await S256K1.schnorr.sign(message, privateKey);
  const risValid = await S256K1.schnorr.verify(rsignature, message, rpub);
})();
```

## API

- [`getPublicKey(privateKey)`](#getpublickeyprivatekey)
- [`sign(msgHash, privateKey)`](#signmsghash-privatekey)
- [`verify(signature, msgHash, publicKey)`](#verifysignature-msghash-publickey)
- [`getSharedSecret(privateKeyA, publicKeyB)`](#getsharedsecretprivatekeya-publickeyb)
- [`recoverPublicKey(hash, signature, recovery)`](#recoverpublickeyhash-signature-recovery)
- [`schnorr.getPublicKey(privateKey)`](#schnorrgetpublickeyprivatekey)
- [`schnorr.sign(message, privateKey)`](#schnorrsignmessage-privatekey)
- [`schnorr.verify(signature, message, publicKey)`](#schnorrverifysignature-message-publickey)
- [Utilities](#utilities)

##### `getPublicKey(privateKey)`
```typescript
function getPublicKey(privateKey: Uint8Array | string | bigint, isCompressed = false): Uint8Array;
```

Creates public key for the corresponding private key.

- `isCompressed = false` determines whether to return compact (33-byte), or full (65-byte) key.

Internally, it does `Point.BASE.multiply(privateKey)`. If you need actual `Point` instead of
`Uint8Array`, use `Point.fromPrivateKey(privateKey)`.

##### `sign(msgHash, privateKey)`
```typescript
function sign(msg: Uint8Array | string, privateKey: Uint8Array | string, opts?: Options): Promise<Uint8Array>;
function sign(msg: Uint8Array | string, privateKey: Uint8Array | string, opts?: Options): Promise<[Uint8Array, number]>;
```

Generates low-s deterministic ECDSA signature as per RFC6979.

- `msgHash: Uint8Array | string` - 32-byte message hash which would be signed
- `privateKey: Uint8Array | string | bigint` - private key which will sign the hash
- `options?: Options` - *optional* object related to signature value and format with following keys:
    - `recovered: boolean = false` - whether the recovered bit should be included in the result. In this case, the result would be an array of two items.
    - `canonical: boolean = true` - whether a signature `s` should be no more than 1/2 prime order.
      `true` (default) makes signatures compatible with libsecp256k1,
      `false` makes signatures compatible with openssl
    - `der: boolean = true` - whether the returned signature should be in DER format. If `false`, it would be in Compact format (32-byte r + 32-byte s)
    - `extraEntropy: Uint8Array | string | true` - additional entropy `k'` for deterministic signature, follows section 3.6 of RFC6979. When `true`, it would automatically be filled with 32 bytes of cryptographically secure entropy. **Strongly recommended** to pass `true` to improve security:
        - Schnorr signatures are doing it every time
        - It would help a lot in case there is an error somewhere in `k` generation. Exposing `k` could leak private keys
        - If the entropy generator is broken, signatures would be the same as they are without the option
        - Signatures with extra entropy would have different `r` / `s`, which means they
        would still be valid, but may break some test vectors if you're cross-testing against other libs

##### `verify(signature, msgHash, publicKey)`
```typescript
function verify(signature: Uint8Array | string, msgHash: Uint8Array | string, publicKey: Uint8Array | string): boolean
function verify(signature: Signature, msgHash: Uint8Array | string, publicKey: Point): boolean
```
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `msgHash: Uint8Array | string` - message hash that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- `options?: Options` - *optional* object related to signature value and format
  - `strict: boolean = true` - whether a signature `s` should be no more than 1/2 prime order.
    `true` (default) makes signatures compatible with libsecp256k1,
    `false` makes signatures compatible with openssl
- Returns `boolean`: `true` if `signature == hash`; otherwise `false`

##### `getSharedSecret(privateKeyA, publicKeyB)`
```typescript
function getSharedSecret(privateKeyA: Uint8Array | string | bigint, publicKeyB: Uint8Array | string | Point, isCompressed = false): Uint8Array;
```

Computes ECDH (Elliptic Curve Diffie-Hellman) shared secret between a private key and a different public key.

- To get Point instance, use `Point.fromHex(publicKeyB).multiply(privateKeyA)`
- `isCompressed = false` determines whether to return compact (33-byte), or full (65-byte) key
- If you have one public key you'll be creating lots of secrets against,
  consider massive speed-up by using precomputations:

    ```js
    const pub = S256K1.utils.precompute(8, publicKeyB);
    // Use pub everywhere instead of publicKeyB
    getSharedSecret(privKey, pub); // Now 12x faster
    ```

##### `recoverPublicKey(hash, signature, recovery)`
```typescript
function recoverPublicKey(msgHash: Uint8Array | string, signature: Uint8Array | string, recovery: number, isCompressed = false): Uint8Array | undefined;
```
- `msgHash: Uint8Array | string` - message hash which would be signed
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `recovery: number` - recovery bit returned by `sign` with `recovered` option
- `isCompressed = false` determines whether to return compact (33-byte), or full (65-byte) key

Public key is generated by doing scalar multiplication of a base Point(x, y) by a fixed
integer. The result is another `Point(x, y)` which we will by default encode to hex Uint8Array.
If signature is invalid - function will return `undefined` as result.
To get Point instance, use `Point.fromSignature(hash, signature, recovery)`.

##### `schnorr.getPublicKey(privateKey)`
```typescript
function schnorrGetPublicKey(privateKey: Uint8Array | string): Uint8Array;
```

Calculates 32-byte public key from a private key.

*Warning:* it is incompatible with non-schnorr pubkey. Specifically, its *y* coordinate may be flipped. See [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) for clarification.

##### `schnorr.sign(message, privateKey)`
```typescript
function schnorrSign(message: Uint8Array | string, privateKey: Uint8Array | string, auxilaryRandom?: Uint8Array): Promise<Uint8Array>;
```

Generates Schnorr signature as per BIP0340. Asynchronous, so use `await`.

- `message: Uint8Array | string` - message (not hash) which would be signed
- `privateKey: Uint8Array | string | bigint` - private key which will sign the hash
- `auxilaryRandom?: Uint8Array` — optional 32 random bytes. By default, the method gathers cryptogarphically secure entropy
- Returns Schnorr signature in Hex format.

##### `schnorr.verify(signature, message, publicKey)`
```typescript
function schnorrVerify(signature: Uint8Array | string, message: Uint8Array | string, publicKey: Uint8Array | string): boolean
```
- `signature: Uint8Array | string | { r: bigint, s: bigint }` - object returned by the `sign` function
- `message: Uint8Array | string` - message (not hash) that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `boolean`: `true` if `signature == hash`; otherwise `false`

#### Utilities

S256K1 exposes a few internal utilities for improved developer experience:

```typescript
const utils: {
  // Can take 40 or more bytes of uniform input e.g. from CSPRNG or KDF
  // and convert them into private key, with the modulo bias being neglible.
  // As per FIPS 186 B.1.1.
  hashToPrivateKey: (hash: Hex) => Uint8Array;
  // Returns `Uint8Array` of 32 cryptographically secure random bytes that can be used as private key
  randomPrivateKey: () => Uint8Array;
  // Checks private key for validity
  isValidPrivateKey(privateKey: PrivKey): boolean;

  // Returns `Uint8Array` of x cryptographically secure random bytes.
  randomBytes: (bytesLength?: number) => Uint8Array;
  // Converts Uint8Array to hex string
  bytesToHex: typeof bytesToHex;
  // Modular division over curve prime
  mod: (number: number | bigint, modulo = CURVE.P): bigint;
  sha256: (message: Uint8Array) => Promise<Uint8Array>;
  hmacSha256: (key: Uint8Array, ...messages: Uint8Array[]) => Promise<Uint8Array>;

  // BIP0340-style tagged hashes
  taggedHash: (tag: string, ...messages: Uint8Array[]) => Promise<Uint8Array>;
  
  // 1. Returns cached point which you can use to pass to `getSharedSecret` or to `#multiply` by it.
  // 2. Precomputes point multiplication table. Is done by default on first `getPublicKey()` call.
  // If you want your first getPublicKey to take 0.16ms instead of 20ms, make sure to call
  // utils.precompute() somewhere without arguments first.
  precompute(size?: number, point?: Point): Point;
};

S256K1.CURVE.P // Field, 2 ** 256 - 2 ** 32 - 977
S256K1.CURVE.n // Order, 2 ** 256 - 432420386565659656852420866394968145599
S256K1.Point.BASE // new secp256k1.Point(Gx, Gy) where
// Gx = 55066263022277343669578718895168534326250603453777594175500187360389116729240n
// Gy = 32670510020758816978083085130507043184471273380659243275938904335757337482424n;

// Elliptic curve point in Affine (x, y) coordinates.
S256K1.Point {
  constructor(x: bigint, y: bigint);
  // Supports compressed and non-compressed hex
  static fromHex(hex: Uint8Array | string);
  static fromPrivateKey(privateKey: Uint8Array | string | number | bigint);
  static fromSignature(
    msgHash: Hex,
    signature: Signature,
    recovery: number | bigint
  ): Point | undefined {
  toRawBytes(isCompressed = false): Uint8Array;
  toHex(isCompressed = false): string;
  equals(other: Point): boolean;
  negate(): Point;
  add(other: Point): Point;
  subtract(other: Point): Point;
  // Constant-time scalar multiplication.
  multiply(scalar: bigint | Uint8Array): Point;
}
S256K1.Signature {
  constructor(r: bigint, s: bigint);
  // DER encoded ECDSA signature
  static fromDER(hex: Uint8Array | string);
  // R, S 32-byte each
  static fromCompact(hex: Uint8Array | string);
  assertValidity(): void;
  hasHighS(): boolean; // high-S sigs cannot be produced using { canonical: true }
  toDERRawBytes(): Uint8Array;
  toDERHex(): string;
  toCompactRawBytes(): Uint8Array;
  toCompactHex(): string;
}
```
