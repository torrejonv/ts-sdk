# Numbers & Points in Elliptic Curve Cryptography

--- BASE THIS ON https://docs.bsvblockchain.org/guides/sdks/ts/low-level/numbers_points -- 

--- IMPROVE WITH Code examples

**Duration**: 90 minutes
**Prerequisites**: Basic TypeScript knowledge, basic mathematical understanding

## Learning Goals
- Understand the mathematical foundations of elliptic curves
- Work with BigInteger numbers in the SDK
- Manipulate elliptic curve points
- Implement point addition and scalar multiplication

## Introduction to Elliptic Curve Mathematics

Elliptic curve cryptography forms the foundation of Bitcoin's cryptographic operations. This tutorial explores the mathematical concepts behind elliptic curves and how to work with them using the BSV TypeScript SDK.

## Setting Up Your Environment

```typescript
import { BigNumber, Point, PrivateKey } from '@bsv/sdk'
```

## Working with BigIntegers

Bitcoin requires handling numbers much larger than JavaScript's native number type can safely represent:

```typescript
// Creating and manipulating large integers
const bigNum1 = new BigNumber(123456789012345678901234567890n)
const bigNum2 = new BigNumber('123456789012345678901234567890')

// Basic operations
const sum = bigNum1.add(bigNum2)
const product = bigNum1.mul(bigNum2)

// Using toHex() instead of toString() for consistent serialization
console.log('Sum:', sum.toHex())
console.log('Product:', product.toHex())
```

## Understanding Elliptic Curves

### The secp256k1 Curve

Bitcoin uses the secp256k1 curve, which has the form y² = x³ + 7 over a finite field:

```typescript
// Constants for the secp256k1 curve
```

## Working with Points on the Curve

### Creating and Manipulating Points

```typescript
// Creating points on the curve
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

// Extracting the underlying point
const point = publicKey.toPoint()

// Point coordinates
const x = point.x
const y = point.y

// Using toHex() instead of toString() for serialization
console.log('Point X coordinate:', x.toHex())
console.log('Point Y coordinate:', y.toHex())
```

### Point Arithmetic

```typescript
// Point addition
const point1 = /* ... */
const point2 = /* ... */
const sumPoint = point1.add(point2)

// Scalar multiplication (key to ECDSA)
const scalar = new BigNumber(123456)
const multipliedPoint = point1.mul(scalar)
```

## The Generator Point

The generator point (G) is a standardized starting point for all calculations on the curve:

```typescript
// Working with the generator point
```

## Practical Applications

### Creating Key Pairs from Scratch

```typescript
// Example of manual key pair generation using curve arithmetic
```

### Verifying Points on the Curve

```typescript
// Example of checking if a point lies on the curve
```

## Advanced Topics

### Finite Field Mathematics

```typescript
// Examples of modular arithmetic operations
```

### Performance Considerations

```typescript
// Optimization techniques for elliptic curve operations
```

## Conclusion

In this tutorial, you've learned about the mathematical foundations of elliptic curve cryptography in Bitcoin. You now understand how to work with large integers, manipulate points on the secp256k1 curve, and perform key cryptographic operations using the BSV TypeScript SDK.

## Further Reading

- [Gentle Introduction to Elliptic Curve Cryptography]()
- [secp256k1 Standard]()
