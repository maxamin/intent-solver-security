# Order Signatures and Their Scope

The signed order is the user's entire protection. Every field must be bound.

## Fields that must be signed

| Field | If missing |
|---|---|
| sell token, buy token | solver swaps different assets |
| sell amount, min buy amount | solver gives worse price |
| receiver | proceeds sent elsewhere |
| deadline / validTo | stale orders filled at bad times |
| nonce / order uid | replay |
| chain ID, settlement contract address (EIP-712 domain) | replay on other chains/contracts |
| fee terms | hidden fees |
| partial fill flag | unexpected partial execution |
| hooks/app data hash | arbitrary pre/post interactions |

"Bound" means more than "present in the struct". The field must be in the **type string**, in the **encoding**, and **enforced** by the settlement logic. A field can be signed but ignored at settlement (for example a `receiver` that the contract reads from calldata instead of the order), which is just as bad as not signing it.

## How EIP-712 hashing can go wrong

The digest is `keccak256("\x19\x01" || domainSeparator || hashStruct(order))`, with `hashStruct = keccak256(typeHash || encodeData(order))`. Each of these can diverge from the intended order:

| Bug | Effect |
|---|---|
| Type string lists a field the encoder omits | signature covers a different struct than the UI shows |
| Dynamic field (`bytes`, `string`) encoded raw instead of hashed | wrong digest, signatures from standard wallets never verify, or collisions |
| Nested struct type not appended to the type string | non-standard hash; wallets display wrong data |
| Domain separator cached at deploy, chain later forks | replay on the forked chain |
| `ecrecover` result not checked against zero | malformed signatures "recover" to address(0) |

## Permit2-based orders

UniswapX uses Permit2 witness signatures: the user signs a Permit2 transfer with the order as "witness". This binds token transfer permission to that specific order. Review that the witness type string and hashing match exactly; mismatches can let a permission be used with a different order.

The witness type string completes Permit2's `PermitWitnessTransferFrom(...)` type stub. Under EIP-712 rules, referenced struct types must be appended after the primary type in alphabetical order. A type string that is close but not exact produces a valid-looking but non-standard digest; wallets that decode typed data then show the user something other than what is verified.

## Phishing angle

Drainers can present malicious "orders" as legitimate (e.g. selling tokens for nothing to an attacker-controlled receiver). Wallets must decode intent orders as clearly as transactions (see **onchain-malware-analysis** Chapter 4).

## Worked example

*Execution status: not executed in this book's CI; verify locally.*

We write a toy order hasher in Solidity, test that every field is bound with a mutation test, and then exploit a version that forgets `receiver`.

**Step 1: the hasher.**

```solidity
struct Order {
    address sellToken; address buyToken; address receiver;
    uint256 sellAmount; uint256 minBuyAmount; uint32 validTo; uint256 nonce;
}

bytes32 constant ORDER_TYPEHASH = keccak256(
    "Order(address sellToken,address buyToken,address receiver,uint256 sellAmount,"
    "uint256 minBuyAmount,uint32 validTo,uint256 nonce)"
);

function hashOrder(Order memory o) public view returns (bytes32) {
    return _hashTypedDataV4(keccak256(abi.encode(
        ORDER_TYPEHASH, o.sellToken, o.buyToken, o.receiver,
        o.sellAmount, o.minBuyAmount, o.validTo, o.nonce
    )));
}
```

`_hashTypedDataV4` comes from OpenZeppelin's `EIP712` base, which builds the domain separator with chain ID and `address(this)` and rebuilds it if the chain ID changes.

**Step 2: the mutation test.**

```solidity
function test_everyFieldBound() public {
    for (uint256 i; i < 7; i++) {
        Order memory o = _baseOrder();
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(userPk, settlement.hashOrder(o));
        if (i == 0) o.sellToken = address(0xBAD);
        if (i == 1) o.buyToken = address(0xBAD);
        if (i == 2) o.receiver = address(0xBAD);
        if (i == 3) o.sellAmount += 1;
        if (i == 4) o.minBuyAmount -= 1;
        if (i == 5) o.validTo += 1;
        if (i == 6) o.nonce += 1;
        address rec = ECDSA.recover(settlement.hashOrder(o), v, r, s);
        assertTrue(rec != user, string.concat("unbound field ", vm.toString(i)));
    }
}
```

Each iteration signs the base order, then changes one field and checks that the signature no longer recovers to the user.

**Step 3: break it.** Remove `o.receiver` from the `abi.encode` (but leave it in the type string, a realistic copy-paste bug). Rerun: the test should fail with `unbound field 2`. Now write the exploit test: a solver takes a user's valid signed order, sets `receiver = solver`, calls `settle`, and the assertion `usdc.balanceOf(user) >= minBuyAmount` fails while the solver's balance rises.

**Step 4: a second, subtler break.** Restore `receiver` in the encoding, but make `settle` pay `msg.sender`'s chosen recipient from calldata instead of `o.receiver`. The mutation test now passes (the hash is fine), yet the exploit still works. That is why Chapter 3's settlement invariant S2 ("each user receives at the signed receiver") is needed in addition to signature tests.

**Step 5: wallet view.** Sign the same order with `cast wallet sign --data` (Chapter 1) and with the Solidity hasher. The two signatures must be identical for the same key. If they differ, your type string or encoding is non-standard, and real wallets will show users something different from what your contract verifies.

## Common pitfalls

- Fields in the struct and type string but not in `abi.encode`, or the reverse.
- Hashing `bytes` and `string` fields raw instead of with `keccak256`, as EIP-712 requires.
- Settlement logic that reads a field from calldata instead of from the signed order.
- Domain separators without `chainId` or `verifyingContract`.
- Witness type strings with nested types out of alphabetical order.
- Treating a passing signature test as proof that settlement honours the fields.

## Exercises

1. Write an EIP-712 order type for a toy intent protocol and test that changing each field invalidates the signature.
2. Build a decoder that displays a UniswapX order in human-readable form.
3. Create a test where the receiver field is omitted from the hash and exploit it.
4. Add a `bytes appData` field and write a test showing the difference between hashing it correctly and encoding it raw.

## Further reading

- EIP-712: <https://eips.ethereum.org/EIPS/eip-712>
- Permit2 witness transfers: <https://github.com/Uniswap/permit2>
- CoW Protocol order structure: <https://docs.cow.fi>
- OpenZeppelin `EIP712` and `ECDSA`: <https://docs.openzeppelin.com/contracts/5.x/api/utils>
