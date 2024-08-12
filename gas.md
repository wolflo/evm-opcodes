# Appendix - Dynamic Gas Costs

### A0-0: Intrinsic Gas
Intrinsic gas is the amount of gas paid prior to execution of a transaction.
That is, the gas paid by the initiator of a transaction, which will always be an externally-owned account, before any state updates are made or any code is executed.

Gas Calculation:
- `gas_cost = 21000`: base cost
- **If** `tx.to == null` (contract creation tx):
    - `gas_cost += 32000`
- `gas_cost += 4 * bytes_zero`: gas added to base cost for every zero byte of memory data
- `gas_cost += 16 * bytes_nonzero`: gas added to base cost for every nonzero byte of memory data

### A0-1: Memory Expansion
An additional gas cost is paid by any operation that expands the memory that is in use.
This memory expansion cost is dependent on the existing memory size and is `0` if the operation does not reference a memory address higher than the existing highest referenced memory address.
A reference is any read, write, or other usage of memory (such as in a `CALL`).

Terms:
- `new_mem_size`: the highest referenced memory address after the operation in question (in bytes)
- `new_mem_size_words = (new_mem_size + 31) // 32`: number of (32-byte) words required for memory after the operation in question
- <code><em>C<sub>mem</sub>(machine_state)</em></code>: the memory cost function for a given machine state

Gas Calculation:
- <code>gas\_cost = <em>C<sub>mem</sub>(new_state)</em> - <em>C<sub>mem</sub>(old_state)</em></code>
- <code>gas\_cost = (new\_mem\_size\_words ^ 2 // 512) + (3 * new\_mem\_size\_words) - <em>C<sub>mem</sub>(old_state)</em></code>

Useful Notes:
- The following opcodes incur a memory expansion cost in addition to their otherwise static gas cost: `RETURN`, `REVERT`, `MLOAD`, `MSTORE`, `MSTORE8`.
- Referencing a zero length range does not require memory to be extended to the beginning of the range.
- The memory cost function is linear up to 724 bytes of memory used, at which point additional memory costs substantially more.

### A0-2: Access Sets
As of [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929), two transaction-wide access sets are maintained.
These access sets keep track of which addresses and storage slots have already been touched within the current transaction.

- `touched_addresses : Set[Address]`
    - a set where every element is an address
    - initialized to include `tx.origin`, `tx.to`\*, and all precompiles
    - \* For a contract creation transaction, `touched_addresses` is initialized to include the address of the created contract instead of `tx.to`, which is the zero address.
- `touched_storage_slots : Set[(Address, Bytes32)]`
    - a set where every element is a tuple, `(address, storage_key)`
    - initialized to the empty set, `{}`

The access sets above are relevant for the following operations:
- `ADDRESS_TOUCHING_OPCODES := { EXTCODESIZE, EXTCODECOPY, EXTCODEHASH, BALANCE, CALL, CALLCODE, DELEGATECALL, STATICCALL, SELFDESTRUCT }`
- `STORAGE_TOUCHING_OPCODES := { SLOAD, SSTORE }`

#### Updating the Access Sets

When an address is the target of one of the `ADDRESS_TOUCHING_OPCODES`, the address is immediately added to the `touched_addresses` set.

- <code>touched addresses = touched_addresses &#x222A; { target_address }</code>

When a storage slot is the target of one of the `STORAGE_TOUCHING_OPCODES`, the `(address, key)` pair is immediately added to the `touched_storage_slots` set.

- <code>touched_storage_slots = touched_storage_slots &#x222A; { (current_address, target_storage_key) }</code>

Important Notes:
- Adding duplicate elements to these sets is a no-op. Performant implementations will use a map with more complicated addition logic.
- If an execution frame reverts, the access sets will return to the state they were in before the frame was entered.
- Additions to the `touched_addresses` set for `*CALL` and `CREATE*` opcodes are made immediately *before* the new execution frames are entered, so any failure within a call or contract creation will not remove the target address of the failing `*CALL` or `CREATE*` from the `touched_addresses` set.
Some tricky edge cases:
    - For `*CALL`, if the call fails for exceeding the maximum call depth or attempting to send more value than the balance of the current address, the target address remains in the `touched_addresses` set.
    - For `CREATE*`, if the contract creation fails for exceeding the maximum call depth or attempting to send more value than the balance of the current address, the address of the created contract does **NOT** remain in the `touched_addresses` set.
    - If a `CREATE*` operation fails for attempting to deploy a contract to a non-empty account, the address of the created contract remains in the `touched_addresses` set.

#### Pre-populating the Access Sets

[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) introduced an optional access *list* that can be included as part of a transaction.
This access list allows elements to be added to the `touched_addresses` and `touched_storage_slots` access *sets* before execution of a transaction begins.
The cost for doing so is `2400` gas for each address added to `touched_addresses` and `1900` gas for each `(address, storage_key)` pair added to `touched_storage_slots`.
This cost is charged at the same time as [intrinsic gas](#a0-0-intrinsic-gas).

Important Notes:
- No `(ADDR, storage_key)` pair can be added to `touched_storage_slots` without also adding `ADDR` to `touched_addresses`.
- The access list may contain duplicates.
The addition of a duplicate item to one of the access sets is a no-op, but the cost of addition is still charged.
- Providing an access list with a transaction can yield a modest discount for each unique access, but this is not always the case.
See [@fvictorio/gas-costs-after-berlin](https://hackmd.io/@fvictorio/gas-costs-after-berlin) for a more complete discussion.

### A0-3: Gas Refunds
Originally intended to provide incentive for clearing unused state, the total `gas_refund` is tracked throughout the execution of a transaction.
As of [EIP-3529](https://eips.ethereum.org/EIPS/eip-3529), `SSTORE` is the only operation that potentially provides a gas refund.

The maximum refund for a transaction is capped at 1/5<sup>th</sup> the gas consumed for the entire transaction.
```
refund_amt := min(gas_refund, tx.gas_consumed // 5)
```
The gas consumed includes the [intrinsic gas](#a0-0-intrinsic-gas), the cost of [pre-populating](#pre-populating-the-access-sets) the access sets, and the cost of any code execution.

When a transaction is finalized, the gas used by the transaction is decreased by `refund_amt`.
This effectively reimburses <code>refund_amt&#160;\*&#160;tx.gasprice</code> to `tx.origin`, but it has the added effect of decreasing the impact of the transaction on the total gas consumed in the block (for purposes of determining the block gas limit).

Because the gas refund is not applied until the end of a transaction, a nonzero refund will not affect whether or not a transaction results in an `OUT_OF_GAS` exception.

#

## A1: EXP
Terms:
- `byte_len_exponent`: the number of bytes in the exponent (exponent is `b` in the stack representation)

Gas Calculation:
- `gas_cost = 10 + 50 * byte_len_exponent`


## A2: SHA3
Terms:
- `data_size`: size of the message to hash in bytes (`len` in the stack representation)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the message to hash
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))

Gas Calculation:
- `gas_cost = 30 + 6 * data_size_words + mem_expansion_cost`


## A3: \*COPY Operations
The following applies for the operations `CALLDATACOPY`, `CODECOPY`, and `RETURNDATACOPY` (not `EXTCODECOPY`).

Terms:
- `data_size`: size of the data to copy in bytes (`len` in the stack representation)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the data to copy
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))

Gas Calculation:
- `gas_cost = 3 + 3 * data_size_words + mem_expansion_cost`


## A4: EXTCODECOPY

Terms:
- `target_addr`: the address to copy code from (`addr` in the stack representation)
- `access_cost`: The cost of accessing a warm vs. cold account (see [A0-2](#a0-2-access-sets))
    - `access_cost = 100` **if** `target_addr` **in** `touched_addresses` (warm access)
    - `access_cost = 2600` **if** `target_addr` **not in** `touched_addresses` (cold access)
- `data_size`: size of the data to copy in bytes (`len` in the stack representation)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the data to copy
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))

Gas Calculation:
- `gas_cost = access_cost + 3 * data_size_words + mem_expansion_cost`


## A5: BALANCE, EXTCODESIZE, EXTCODEHASH

The opcodes `BALANCE`, `EXTCODESIZE`, `EXTCODEHASH` have the same pricing function based on making a single account access.
See [A0-2](#a0-2-access-sets) for details on EIP-2929 and `touched_addresses`.

Terms:
- `target_addr`: the address of interest (`addr` in the opcode stack representations)

Gas Calculation:
- `gas_cost = 100` **if** `target_addr` **in** `touched_addresses` (warm access)
- `gas_cost = 2600` **if** `target_addr` **not in** `touched_addresses` (cold access)


## A6: SLOAD

See [A0-2](#a0-2-access-sets) for details on EIP-2929 and `touched_storage_slots`.

Terms:
- `context_addr`: the address of the current execution context (i.e. what `ADDRESS` would put on the stack)
- `target_storage_key`: The 32-byte storage index to load from (`key` in the stack representation)

Gas Calculation:
- `gas_cost = 100` **if** `(context_addr, target_storage_key)` **in** `touched_storage_slots` (warm access)
- `gas_cost = 2100` **if** `(context_addr, target_storage_key)` **not in** `touched_storage_slots` (cold access)


## A7: SSTORE
This gets messy.
See [EIP-2200](https://eips.ethereum.org/EIPS/eip-2200), activated in the Istanbul hardfork.

The cost of an `SSTORE` operation is dependent on the existing value and the value to be stored:
1. Zero vs. nonzero values - storing nonzero values is more costly than storing zero
1. The current value of the slot vs. the value to store - changing the value of a slot is more costly than not changing it
2. "Dirty" vs. "clean" slot - changing a slot that has not yet been changed within the current execution context is more costly than changing a slot that has already been changed

The cost is also dependent on whether or not the targeted storage slot has already been accessed within the same transaction.
See [A0-2](#a0-2-access-sets) for details on EIP-2929 and `touched_storage_slots`.

Terms:
- `context_addr`: the address of the current execution context (i.e. what `ADDRESS` would put on the stack)
- `target_storage_key`: The 32-byte storage index to store to (`key` in the stack representation)
- `orig_val`: the value of the storage slot if the current transaction is reverted
- `current_val`: the value of the storage slot immediately *before* the `sstore` op in question
- `new_val`: the value of the storage slot immediately *after* the `sstore` op in question

Gas Calculation:
- `gas_cost = 0`
- `gas_refund = 0`
- **If** `gas_left <= 2300`:
    - `throw OUT_OF_GAS_ERROR` (can not `sstore` with < 2300 gas for backwards compatibility)
- **If** `(context_addr, target_storage_key)` **not in** `touched_storage_slots` ([cold access](#a0-2-access-sets)):
    - `gas_cost += 2100`
- **If** `new_val == current_val` (no-op):
    - `gas_cost += 100`
- **Else** `new_val != current_val`:
    - **If** `current_val == orig_val` ("clean slot", not yet updated in current execution context):
        - **If** `orig_val == 0` (slot started zero, currently still zero, now being changed to nonzero):
            - `gas_cost += 20000`
        - **Else** `orig_val != 0` (slot started nonzero, currently still same nonzero value, now being changed):
            - `gas_cost += 2900` and update the refund as follows..
            - **If** `new_val == 0` (the value to store is 0):
                - `gas_refund += 4800`
    - **Else** `current_val != orig_val` ("dirty slot", already updated in current execution context):
        - `gas_cost += 100` and update the refund as follows..
        - **If** `orig_val != 0` (execution context started with a nonzero value in slot):
            - **If** `current_val == 0` (slot started nonzero, currently zero, now being changed to nonzero):
                - `gas_refund -= 4800`
            - **Else if** `new_val == 0` (slot started nonzero, currently still nonzero, now being changed to zero):
                - `gas_refund += 4800`
        - **If** `new_val == orig_val` (slot is reset to the value it started with):
            - **If** `orig_val == 0` (slot started zero, currently nonzero, now being reset to zero):
                - `gas_refund += 19900`
            - **Else** `orig_val != 0` (slot started nonzero, currently different nonzero value, now reset to orig. nonzero value):
                - `gas_refund += 2800`


## A8: LOG\* Operations
Note that for `LOG*` operations gas is paid per byte of data (not per word).

Terms:
- `num_topics`: the \* of the LOG\* op. e.g. LOG0 has `num_topics = 0`, LOG4 has `num_topics = 4`
- `data_size`: size of the data to log in bytes (`len` in the stack representation).
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))

Gas Calculation:
- `gas_cost = 375 + 375 * num_topics + 8 * data_size + mem_expansion_cost`


## A9: CREATE\* Operations

Common Terms:
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))
- `code_deposit_cost`: the per-byte cost incurred for storing the deployed code (see [A9-F](#a9-f-code-deposit-cost)).

### A9-1: CREATE

Gas Calculation:
- `gas_cost = 32000 + mem_expansion_cost + code_deposit_cost`

### A9-2: CREATE2
`CREATE2` incurs an additional dynamic cost over `CREATE` because of the need to hash the init code.

Terms:
- `data_size`: size of the init code in bytes (`len` in the stack representation)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the init code

Gas Calculation:
- `gas_cost = 32000 + 6 * data_size_words + mem_expansion_cost + code_deposit_cost`

### A9-F: Code Deposit Cost
In addition to the static and dynamic cost of the `CREATE` and `CREATE2` operations, a per-byte cost is charged for storing the returned runtime code.
Unlike the static and dynamic costs of the opcodes, this cost is not applied until *after* the execution of the initialization code halts.

Terms:
- `returned_code_size`: the length of the returned runtime code

Gas Calculation:
- `code_deposit_cost = 200 * returned_code_size`

A note related to the code deposit step of contract creation:
As of [EIP-3541](https://eips.ethereum.org/EIPS/eip-3541), any code returned from a contract creation (i.e. what *would* become the deployed contract code), results in an exceptional abort of the entire contract creation if the code's first byte is `0xEF`.

## AA: CALL\* Operations
Gas costs for `CALL`, `CALLCODE`, `DELEGATECALL`, and `STATICCALL` operations.
A big piece of the gas calculation for these operations is determining the gas to send along with the call.
There's a good chance that you are primarily interested in the `base_cost` and can ignore this additional calculation, because the `gas_sent_with_call` is consumed in the context of the called contract, and the unconsumed gas is returned.
If not, see the `gas_sent_with_call` [section](#aa-f-gas-to-send-with-call-operations).

Similar to selfdestruct, `CALL` incurs an additional cost if it forces an account to be added to the state tree by sending a nonzero amount of eth to an address that was previously empty.
"Empty", in this case is defined according to [EIP-161](https://eips.ethereum.org/EIPS/eip-161) (`balance == nonce == code == 0x`).

Common Terms:
- `call_value`: the value sent with the call (`val` in the stack representation)
- `target_addr`: the recipient of the call (`addr` in the stack representation)
- `access_cost`: The cost of accessing a warm vs. cold account (see [A0-2](#a0-2-access-sets))
    - `access_cost = 100` **if** `target_addr` **in** `touched_addresses` (warm access)
    - `access_cost = 2600` **if** `target_addr` **not in** `touched_addresses` (cold access)
- `mem_expansion_cost`: the cost of any memory expansion required (see [A0-1](#a0-1-memory-expansion))
- `gas_sent_with_call`: the gas ultimately sent with the call

### AA-1: CALL

Gas Calculation:
- `base_gas = access_cost + mem_expansion_cost`
- **If** `call_value > 0` (sending value with call):
    - `base_gas += 9000`
    - **If** `is_empty(target_addr)` (forcing a new account to be created in the state trie):
        - `base_gas += 25000`

Calculate the `gas_sent_with_call` [below](#aa-f-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AA-2: CALLCODE

Gas Calculation:
- `base_gas = access_cost + mem_expansion_cost`
- **If** `call_value > 0` (sending value with call):
    - `base_gas += 9000`

Calculate the `gas_sent_with_call` [below](#aa-f-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AA-3: DELEGATECALL

Gas Calculation:
- `base_gas = access_cost + mem_expansion_cost`

Calculate the `gas_sent_with_call` [below](#aa-f-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AA-4: STATICCALL

Gas Calculation:
- `base_gas = access_cost + mem_expansion_cost`

Calculate the `gas_sent_with_call` [below](#aa-f-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AA-F: Gas to Send with CALL Operations
In addition to the base cost of executing the operation, `CALL`, `CALLCODE`, `DELEGATECALL`, and `STATICCALL` need to determine how much gas to send along with the call.
Much of the complexity here comes from a backward-compatible change made in [EIP-150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md).
Here's an overview of why this calculation is used:

EIP-150 increased the `base_cost` of the `CALL` opcode from 40 to 700 gas, but most contracts in use at the time were sending `available_gas - 40` with every call.
So, when `base_cost` increased, these contracts were suddenly trying to send more gas than they had left (`requested_gas > remaining_gas`).
To avoid breaking these contracts, if the `requested_gas` is more than `remaining_gas`, we send `all_but_one_64th` of `remaining_gas` instead of trying to send `requested_gas`, which would result in an `OUT_OF_GAS_ERROR`.

Terms:
- `base_gas`: the cost of the operation before taking into account the gas that should be sent along with the call.
See [AA](#aa-call-operations) for this calculation.
- `available_gas`: the gas remaining in the current execution context immediately before the execution of the opcode
- `remaining_gas`: the gas remaining after deducting `base_cost` of the op but before deducting `gas_sent_with_call`
- `requested_gas`: the gas requested to be sent with the call (`gas` in the stack representation)
- `all_but_one_64th`: All but floor(1/64) of the remaining gas
- `gas_sent_with_call`: the gas ultimately sent with the call

Gas Calculation:
- `remaining_gas = available_gas - base_gas`
- `all_but_one_64th = remaining_gas - (remaining_gas // 64)`
- `gas_sent_with_call = min(requested_gas, all_but_one_64th)`

Any portion of `gas_sent_with_call` that is not used by the recipient of the call is refunded to the caller after the call returns. Also, if `call_value > 0`, a 2300 gas stipend is added to the amount of gas included in the call, but not to the cost of making the call.


## AB: SELFDESTRUCT
The gas cost of a `SELFDESTRUCT` operation is dependent on whether or not the operation results in a new account being added to the state trie.
If a nonzero amount of eth is sent to an address that was previously empty, an additional cost is incurred.
"Empty", in this case is defined according to [EIP-161](https://eips.ethereum.org/EIPS/eip-161) (`balance == nonce == code == 0x`).

The cost also increases if the operation requires a cold access of the recipient address.
See [A0-2](#a0-2-access-sets) for details on EIP-2929 and `touched_addresses`.

Terms:
- `target_addr`: the recipient of the self-destructing contract's funds (`addr` in the stack representation)
- `context_addr`: the address of the current execution context (i.e. what `ADDRESS` would put on the stack)

Gas Calculation:
- `gas_cost = 5000`: base cost
- **If** `balance(context_addr) > 0 && is_empty(target_addr)` (sending funds to a previously empty address):
    - `gas_cost += 25000`
- **If** `target_addr` **not in** `touched_addresses` (cold access):
    - `gas_cost += 2600`


## AF: INVALID
On execution of any invalid operation, whether the designated `INVALID` opcode or simply an undefined opcode, all remaining gas is consumed and the state is reverted to the point immediately prior to the beginning of the current execution context.
