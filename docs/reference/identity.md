# API

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

## Interfaces

| |
| --- |
| [DisplayableIdentity](#interface-displayableidentity) |
| [IdentityClientOptions](#interface-identityclientoptions) |

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---

### Interface: DisplayableIdentity

```ts
export interface DisplayableIdentity {
    name: string;
    avatarURL: string;
    abbreviatedKey: string;
    identityKey: string;
    badgeIconURL: string;
    badgeLabel: string;
    badgeClickURL: string;
}
```

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
### Interface: IdentityClientOptions

```ts
export interface IdentityClientOptions {
    protocolID: WalletProtocol;
    keyID: string;
    tokenAmount: number;
    outputIndex: number;
}
```

See also: [WalletProtocol](./wallet.md#type-walletprotocol)

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
## Classes

### Class: IdentityClient

IdentityClient lets you discover who others are, and let the world know who you are.

```ts
export class IdentityClient {
    constructor(wallet?: WalletInterface, private readonly options = DEFAULT_IDENTITY_CLIENT_OPTIONS, private readonly originator?: OriginatorDomainNameStringUnder250Bytes) 
    async publiclyRevealAttributes(certificate: WalletCertificate, fieldsToReveal: CertificateFieldNameUnder50Bytes[]): Promise<BroadcastResponse | BroadcastFailure> 
    async resolveByIdentityKey(args: DiscoverByIdentityKeyArgs): Promise<DisplayableIdentity[]> 
    async resolveByAttributes(args: DiscoverByAttributesArgs): Promise<DisplayableIdentity[]> 
    static parseIdentity(identityToParse: IdentityCertificate): DisplayableIdentity 
}
```

See also: [BroadcastFailure](./transaction.md#interface-broadcastfailure), [BroadcastResponse](./transaction.md#interface-broadcastresponse), [CertificateFieldNameUnder50Bytes](./wallet.md#type-certificatefieldnameunder50bytes), [DEFAULT_IDENTITY_CLIENT_OPTIONS](./identity.md#variable-default_identity_client_options), [DiscoverByAttributesArgs](./wallet.md#interface-discoverbyattributesargs), [DiscoverByIdentityKeyArgs](./wallet.md#interface-discoverbyidentitykeyargs), [DisplayableIdentity](./identity.md#interface-displayableidentity), [IdentityCertificate](./wallet.md#interface-identitycertificate), [OriginatorDomainNameStringUnder250Bytes](./wallet.md#type-originatordomainnamestringunder250bytes), [WalletCertificate](./wallet.md#interface-walletcertificate), [WalletInterface](./wallet.md#interface-walletinterface)

#### Method parseIdentity

TODO: Implement once revocation overlay is created
Remove public certificate revelation from overlay services by spending the identity token

Parse out identity and certifier attributes to display from an IdentityCertificate

```ts
static parseIdentity(identityToParse: IdentityCertificate): DisplayableIdentity 
```
See also: [DisplayableIdentity](./identity.md#interface-displayableidentity), [IdentityCertificate](./wallet.md#interface-identitycertificate)

Returns

- IdentityToDisplay

Argument Details

+ **serialNumber**
  + Unique serial number of the certificate to revoke revelation
+ **identityToParse**
  + The Identity Certificate to parse

#### Method publiclyRevealAttributes

Publicly reveals selected fields from a given certificate by creating a publicly verifiable certificate.
The publicly revealed certificate is included in a blockchain transaction and broadcast to a federated overlay node.

```ts
async publiclyRevealAttributes(certificate: WalletCertificate, fieldsToReveal: CertificateFieldNameUnder50Bytes[]): Promise<BroadcastResponse | BroadcastFailure> 
```
See also: [BroadcastFailure](./transaction.md#interface-broadcastfailure), [BroadcastResponse](./transaction.md#interface-broadcastresponse), [CertificateFieldNameUnder50Bytes](./wallet.md#type-certificatefieldnameunder50bytes), [WalletCertificate](./wallet.md#interface-walletcertificate)

Returns

A promise that resolves with the broadcast result from the overlay network.

Argument Details

+ **certificate**
  + The master certificate to selectively reveal.
+ **fieldsToReveal**
  + An array of certificate field names to reveal. Only these fields will be included in the public certificate.

Throws

Throws an error if the certificate is invalid, the fields cannot be revealed, or if the broadcast fails.

#### Method resolveByAttributes

Resolves displayable identity certificates by specific identity attributes, issued by a trusted entity.

```ts
async resolveByAttributes(args: DiscoverByAttributesArgs): Promise<DisplayableIdentity[]> 
```
See also: [DiscoverByAttributesArgs](./wallet.md#interface-discoverbyattributesargs), [DisplayableIdentity](./identity.md#interface-displayableidentity)

Returns

The promise resolves to displayable identities.

Argument Details

+ **args**
  + Attributes and optional parameters used to discover certificates.

#### Method resolveByIdentityKey

Resolves displayable identity certificates, issued to a given identity key by a trusted certifier.

```ts
async resolveByIdentityKey(args: DiscoverByIdentityKeyArgs): Promise<DisplayableIdentity[]> 
```
See also: [DiscoverByIdentityKeyArgs](./wallet.md#interface-discoverbyidentitykeyargs), [DisplayableIdentity](./identity.md#interface-displayableidentity)

Returns

The promise resolves to displayable identities.

Argument Details

+ **args**
  + Arguments for requesting the discovery based on the identity key.

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
## Functions

## Types

## Enums

## Variables

| |
| --- |
| [DEFAULT_IDENTITY_CLIENT_OPTIONS](#variable-default_identity_client_options) |
| [KNOWN_IDENTITY_TYPES](#variable-known_identity_types) |
| [defaultIdentity](#variable-defaultidentity) |

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---

### Variable: DEFAULT_IDENTITY_CLIENT_OPTIONS

```ts
DEFAULT_IDENTITY_CLIENT_OPTIONS: IdentityClientOptions = {
    protocolID: [1, "identity"],
    keyID: "1",
    tokenAmount: 1,
    outputIndex: 0
}
```

See also: [IdentityClientOptions](./identity.md#interface-identityclientoptions)

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
### Variable: KNOWN_IDENTITY_TYPES

```ts
KNOWN_IDENTITY_TYPES = {
    identiCert: "z40BOInXkI8m7f/wBrv4MJ09bZfzZbTj2fJqCtONqCY=",
    discordCert: "2TgqRC35B1zehGmB21xveZNc7i5iqHc0uxMb+1NMPW4=",
    phoneCert: "mffUklUzxbHr65xLohn0hRL0Tq2GjW1GYF/OPfzqJ6A=",
    xCert: "vdDWvftf1H+5+ZprUw123kjHlywH+v20aPQTuXgMpNc=",
    registrant: "YoPsbfR6YQczjzPdHCoGC7nJsOdPQR50+SYqcWpJ0y0=",
    emailCert: "exOl3KM0dIJ04EW5pZgbZmPag6MdJXd3/a1enmUU/BA=",
    anyone: "mfkOMfLDQmrr3SBxBQ5WeE+6Hy3VJRFq6w4A5Ljtlis=",
    self: "Hkge6X5JRxt1cWXtHLCrSTg6dCVTxjQJJ48iOYd7n3g=",
    coolCert: "AGfk/WrT1eBDXpz3mcw386Zww2HmqcIn3uY6x4Af1eo="
}
```

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
### Variable: defaultIdentity

```ts
defaultIdentity: DisplayableIdentity = {
    name: "Unknown Identity",
    avatarURL: "XUUB8bbn9fEthk15Ge3zTQXypUShfC94vFjp65v7u5CQ8qkpxzst",
    identityKey: "",
    abbreviatedKey: "",
    badgeIconURL: "XUUV39HVPkpmMzYNTx7rpKzJvXfeiVyQWg2vfSpjBAuhunTCA9uG",
    badgeLabel: "Not verified by anyone you trust.",
    badgeClickURL: "https://projectbabbage.com/docs/unknown-identity"
}
```

See also: [DisplayableIdentity](./identity.md#interface-displayableidentity)

Links: [API](#api), [Interfaces](#interfaces), [Classes](#classes), [Functions](#functions), [Types](#types), [Enums](#enums), [Variables](#variables)

---
