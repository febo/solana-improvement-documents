---
simd: 'XXXX'
title: `p-token` – Efficient Token program
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Draft
created: 2025-01-26
feature: (fill in with feature tracking issues once accepted)
supersedes: (optional - fill this in if the SIMD supersedes a previous SIMD)
superseded-by: (optional - fill this in if the SIMD is superseded by a subsequent
 SIMD)
extends: (optional - fill this in if the SIMD extends the design of a previous
 SIMD)
---

## Summary

Replace the current version of SPL Token (`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`) program by a CU-optimized one (`p-token`).

## Motivation

About `~10%` of block compute units is used by the Token program instructions. Decreasing the CUs used by Token program instructions creates block space for other transactions to be executed – i.e., less CUs consumed by Token program, more CUs for other transactions.

As an example, if we reduce the CUs consumed by Token instructions to `1/20th` of their current value, `10%` of block CUs utilized becomes `0.5%`, resulting in a `9.5%` block space gain.

Additionally, there are benefits to downstream programs:
* Better composability since using the Token program instructions require less CUs.
* Cheaper (in terms of CU) cross-program invocations.

## New Terminology

N/A.

## Detailed Design

`p-token` is a like-for-like efficient re-implementation of the Token program. It is `no_std` (no heap memory allocations are made in the program) and uses zero-copy access for instruction and account data. Since it follows the same instructions and accounts layout, it does not require any changes to client code – it works as a drop-in replacement.

Apart from the original SPL Token instructions, this proposal adds two additional instructions to the program:

1. `withdraw_excess_lamports` (instruction discriminator `38`): allow recovering "bricked" SOL from mint accounts (e.g., USDC mint as `~323` SOL in excess). The logic of this instruction is the same as the current SPL Token-2022 instruction. 
2. `batch` (instruction discriminator `255`): enable efficient CPI interaction with the Token program. This is a new instruction that can execute a variable number on Token instructions in a single invocation of the Token program. Therefore, the CPI invoke units (currently `1000` CU) are only consumed once, instead of each CPI instruction – this significantly improves the CUs required to perform multiple Token instructions in a CPI context.

Note that `withdraw_excess_lamports` discriminator matches the value used in SPL Token-2022, while `batch` has a driscriminator value that is not used in either SPL Token nor Token-2022.

The program and its program data will be loaded into accounts, `$PTOKEN_PROGRAM_ACCOUNT` and `$PTOKEN_PROGRAM_DATA_ACCOUNT` respectively, prior to enablind the feature gate that triggers the replacement.

When the feature gate `XXXXXXXXXXXXXX` is enabled, the runtime needs to:

1. Replace `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA` program with `$PTOKEN_PROGRAM_ACCOUNT` using the upgradable loader.
2. Replace the contents of `$TOKENKEG_PROGRAM_DATA_ADDRESS` with the contents of `$PTOKEN_PROGRAM_DATA_ACCOUNT`. The `$TOKENKEG_PROGRAM_DATA_ADDRESS` address is defined as `PDA([TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA], upgradeable_loader_id)`.


## Alternatives Considered

As an alternative to replace the current version of SPL Token, `p-token` could be deployed to a new address and people can be encouraged to transition to that program. This would hinder its adoption and benefits, since people would be very slow to adopt the new program, if they adopt it at all.

## Impact

The main impact is freeing up block CUs, allowing more transactions to be packed in a block; dapp developers benefit since interacting with the Token program will consume significantly less CUs.

Below is a sample of the CU efficient gained by `p-token` compared to the current SPL Token program.

| Instruction                | CU (`p-token`) | CU (`spl-token`) |
|----------------------------|----------------|------------------|
| `InitializeMint`           | 100            | 2967             |
| `InitializeAccount`        | 170            | 4527             |
| `InitializeMultisig`       | 190            | 2973             |
| `Transfer`                 | 153            | 4645             |
| `Approve`                  | 124            | 2904             |
| `Revoke`                   | 97             | 2677             |
| `SetAuthority`             | 126            | 3167             |
| `MintTo`                   | 154            | 4538             |
| `Burn`                     | 168            | 4753             |
| `CloseAccount`             | 147            | 2916             |
| `FreezeAccount`            | 136            | 4265             |
| `ThawAccount`              | 136            | 4267             |

## Security Considerations

`p-token` must be guaranteed to follow the same instructions and accounts layout, as well as have the same behaviour than the current Token implementation.

Any potential risk will be mitigated by extensive fixture testing, formal verification and audits. Since there are potentially huge economic consequences of this change, the feature will be put to a validator vote.

The replacement of the program requires breaking consensus on a non-native program. However, this has been done in the past many times for SPL Token to fix bugs and add new features.

## Drawbacks (Optional)

N/A.

## Backwards Compatibility (Optional)

Fully backwards compatible, no changes are required for users of the program.
