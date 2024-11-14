---
simd: '0XXX'
title: Deprecate Rent Exempt Threshold
authors:
category: Standard
type: Core
status: Review
created: 2024-11-14
feature:
supersedes:
superseded-by:
extends:
---

## Summary

This proposal aims to eliminate the use of floating-point operations to 
determine whether an account is rent exempt or not in programs by deprecating
the use of the `exempt_threshold` value &mdash; the `exempt_threshold` is
currently specified as a `f64`. 

More specifically, it proposes to rely only on the `lamports_per_byte_year` (to 
be renamed to `lamports_per_byte`), which is a `u64` value, doubling the current
value of `lamports_per_byte_year` and set the  `exemption_threshold` to `1.0`,
deprecating its use in SDKs and onchain programs moving forward without 
affecting the existing API.

## Motivation

Many programs validate whether an account is rent-exempt during initialization. 
For instance, the SPL Token program includes specific checks in its 
`initialize_mint` and `initialize_account` instructions to ensure that accounts 
being initialized are rent-exempt. The calculation for determining the minimum 
lamports balance required for an account to achieve rent exemption involves the 
`exemption_threshold` from the Rent sysvar. Currently, this value is 
represented as a `f64`, which requires floating-point arithmetic. 
Floating-point operations are known to be computationally expensive in terms of 
compute units since they are emulated by the VM. Given all accounts are now 
required to be rent-exempt, and there are plans to eventually eliminate the 
concept of "rent" altogether, the use of `f64` is sub-optimal and incurs on an 
unnecessary "penalty" for programs that require its use.

The calculation of the minimum rent exempt lamports consumes approximately 
`~256` compute units. On an optimized token program, this represents about half 
of the compute units required by an instruction that initializes a mint or 
token account. The same calculation without using the `exemtion_threshold`, 
which means without a floating-point operation, consumes `8` compute units.

## New Terminology

None.

## Detailed Design

The minimum lamports balance required for an account to be rent exempt is 
determined by:
```
((ACCOUNT_STORAGE_OVERHEAD + bytes) * lamports_per_byte_year) * 
exemption_threshold
```
where both `lamports_per_byte_year` and `exemption_threshold` are values from 
the Rent sysvar. Since `exemption_threshold` is a `f64`, all values need to 
first be "converted" to `f64` to perform the calculation and the final result 
"converted" back to `u64`. The conversion and the operation with floating-point
values is expensve in the VM.

The proposal is to update the value of `lamports_per_byte_year` by 
`lamports_per_byte_year * exemption_threshold` using their current values and 
set `exemption_threshold` to `1.0`. The `lamports_per_byte_year` will be then 
renamed to `lamports_per_byte` to reflect the new value it represents. The
current `exemption_threshold` value is `2` and `lamports_per_byte_year` is
`3480`. With the changes proposed, `exemption_threshold` will be set to
`1.0` and `lamports_per_byte` to `6960`.

This effectively allows us to deprecate the use of `exemption_threshold` in a 
backwards compatible way, and change the way the minimum lamports requirement 
is calculated to:
```
(ACCOUNT_STORAGE_OVERHEAD + bytes) * lamports_per_byte
```

The benefit is that the calculation now can be efficiently done with only `u64` 
values. At the same time, the previous calculation also evaluates to the same 
value without requiring any changes.

Validator implementations should stop using `exemption_threshold` and only use
the `lamports_per_byte`.

## Alternatives Considered

* Do nothing
  - Doing nothing means that calculating rent exemption on a program remains an 
expensive operation.

* Perform the "conversion" ouside the VM
  - An alternative design would be to change the type of `exemption_threshold` 
to a `u64` using a new `RentV2` struct. This would require reading the 
"original" value as a `f64` and converting it to `u64` outside the VM. For this 
to work, a new syscall and sysvar would need to be defined.

## Impact

This change will significantly improve the compute unit (CU) consumption of
programs that calculate the minimum lamports balance required for rent 
exemption. 
Currently, the calculation of the minimum balance for rent exemption 
(`Rent::minimum_balance`) consumes approximately `256` CUs. With the proposed
change, this will be reduced to just `8` CUs.

## Security Considerations

None.

## Backwards Compatibility

This proposal is backwards compatible with the current protocol, since it does 
not change the absolute value required for an account to be rent exempt and the 
current way of calculating the threshold will continue evaluate to the same 
value. Therefore, any deployed program will be unaffected.