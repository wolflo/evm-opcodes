# Appendix - Dynamic Gas Costs

## A1: Intrinsic Gas
Intrinsic gas is the amount of gas paid prior to execution of a transaction.
That is, the gas paid by the initiator of a transaction, which will always be an externally-owned account, before any state updates are made or any code is executed.

Gas Calculation:
- `gas_cost = 21000`: base cost
- **If** `msg.to == null` (contract creation tx):
    - `gas_cost += 32000`
- `gas_cost += 4 * bytes_zero`: gas added to base cost for every zero byte of memory data
- `gas_cost += 16 * bytes_nonzero`: gas added to base cost for every nonzero byte of memory data

## A2: Memory Expansion
An additional gas cost is paid by any operation that expands the memory that is in use.
This memory expansion cost is dependent on the existing memory size and is `0` if the op does not reference a memory address higher than the existing highest referenced memory address.
A reference is any read, write, or other usage of memory (such as in a `CALL`).

Terms:
- `new_mem_size`: the highest referenced memory address after the operation in question (in bytes)
- `new_mem_size_words = (new_mem_size + 31) // 32`: number of (32-byte) words required for memory after the operation in question
- <code><em>C<sub>mem</sub>(machine_state)</em></code>: the memory cost function for a given machine state

Gas Calculation:
- <code>gas\_cost = <em>C<sub>mem</sub>(new_state)</em> - <em>C<sub>mem</sub>(old_state)</em></code>
- <code>gas\_cost = (new\_mem\_size\_words ^ 2 / 512) + (3 * new\_mem\_size\_words) - <em>C<sub>mem</sub>(old_state)</em></code>

Useful Notes:
- The following ops incur a memory expansion cost in addition to their otherwise static gas cost: `RETURN`, `REVERT`, `MLOAD`, `MSTORE`, `MSTORE8`, `CREATE`.
- Referencing a zero length range does not require memory to be extended to the beginning of the range.
- The memory cost function is linear up to 724 bytes of memory used, at which point additional memory costs substantially more.

## A3: EXP
Terms:
- `byte_len_exponent`: the number of bytes in the exponent (exponent is `b` in the stack representation above)

Gas Calculation:
- `gas_cost = 10 + 50 * byte_len_exponent`

## A4: SHA3
Terms:
- `data_size`: size of the message to hash in bytes (`len` in the stack representation above)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the message to hash
- `mem_expansion_cost`: the cost of any memory expansion required (see [A2](#a2-memory-expansion))

Gas Calculation:
- `gas_cost = 30 + 6 * data_size_words + mem_expansion_cost`

## A5: \*COPY Operations
The following applies for the operations `CALLDATACOPY`, `CODECOPY`, `EXTCODECOPY`, and `RETURNDATACOPY`.
Note that `EXTCODECOPY` has a different `static_cost` than the other ops and also takes its `len` parameter from a different index on the stack.

Terms:
- `static_cost`: the constant base cost for the operation
    - `static_cost = 3` for `CALLDATACOPY`, `CODECOPY`, `RETURNDATACOPY`
    - `static_cost = 700` for `EXTCODECOPY`
- `data_size`: size of the data to copy in bytes (`len` in the stack representation above)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the data to copy
- `mem_expansion_cost`: the cost of any memory expansion required (see [A2](#a2-memory-expansion))

Gas Calculation:
- `gas_cost = static_cost + 3 * data_size_words + mem_expansion_cost`


## A6: SSTORE
This gets messy.
See [EIP-2200](https://eips.ethereum.org/EIPS/eip-2200), implemented in Istanbul hardfork.
The short version is the cost of an `SSTORE` op is dependent on:
1. Zero vs. nonzero values - storing nonzero values is more costly than storing zero
1. The current value of the slot vs. the value to store - changing the value of a slot is more costly than not changing it
2. "Dirty" vs. "clean" slot - changing a slot that has not yet been changed within the current execution context is more costly than changing a slot that has already been changed

Terms:
- `og_val`: the value of the storage slot if the current transaction is reverted
- `current_val`: the value of the storage slot immediately *before* the `sstore` op in question
- `new_val`: the value of the storage slot immediately *after* the `sstore` op in question

Gas Calculation:
- **If** `gas_left <= 2300`:
    - `throw OUT_OF_GAS` (can not `sstore` with < 2300 gas for backwards compatibility)
- `gas_cost = 0`
- `gas_refund = 0`
- **If** `new_val == current_val` (no-op):
    - `gas_cost += 800`
- **If** `new_val != current_val`:
    - **If** `current_val == og_val` ("clean slot", not yet updated in current execution context):
        - **If** `og_val == 0` (slot started zero, currently still zero, now being changed to nonzero):
            - `gas_cost += 20000`
        - **Else** (slot started nonzero, currently still same nonzero value, now being changed):
            - `gas_cost += 5000` and..
            - **If** `new_val == 0` (the value to store is 0):
                - `gas_refund += 15000`
    - **If**: `current_val != og_val` ("dirty slot", already updated in current execution context):
        - `gas_cost += 800` and..
       - **If** `og_val != 0` (execution context started with a nonzero value in slot):
            - **If** `current_val == 0` (slot started nonzero, currently zero, now being changed to nonzero):
                - `gas_refund -= 15000`
            - **If** `new_value == 0` (slot started nonzero, currently still nonzero, now being changed to zero):
                - `gas_refund += 15000`
        - **If** `new_val == og_val` (slot is reset to the value it started with):
            - **If** `og_val == 0` (slot started zero, currently nonzero, now being reset to zero):
                - `gas_refund += 19200`
            - **Else** (slot started nonzero, currently different nonzero value, now reset to orig. nonzero value):
                - `gas_refund += 4200`

## A7: LOG\* Operations
Note that for `LOG*` ops gas is paid per byte of data (not per word).

Terms:
- `num_topics`: the \* of the LOG\* op. e.g. LOG0 has `num_topics = 0`, LOG4 has `num_topics = 4`
- `data_size`: size of the data to log in bytes (`len` in the stack representation above).
- `mem_expansion_cost`: the cost of any memory expansion required (see [A2](#a2-memory-expansion))

Gas Calculation:
- `gas_cost = 375 + 375 * num_topics + 8 * data_size + mem_expansion_cost`


## A9: CREATE2
`CREATE2` incurs an additional dynamic cost over `CREATE` because of the need to hash the init code.
Also note that there is a cost incurred for storing code in addition to the costs presented here for `CREATE` and `CREATE2`.

Terms:
- `data_size`: size of the init code in bytes (`len` in the stack representation above)
- `data_size_words = (data_size + 31) // 32`: number of (32-byte) words in the init code
- `mem_expansion_cost`: the cost of any memory expansion required (see [A2](#a2-memory-expansion))

Gas Calculation:
- `gas_cost = 32000 + 6 * data_size_words + mem_expansion_cost`


## AA: SELFDESTRUCT
The gas cost of a `SELFDESTRUCT` op is dependent on whether or not the op results in a new account being added to the state trie.
If a nonzero amount of eth is sent to an address that was previously empty, an additional cost is incurred.
"Empty", in this case is defined according to [EIP-161](https://eips.ethereum.org/EIPS/eip-161) (`balance == nonce == code == 0x`).

Terms:
- `target_addr`: the recipient of the self-destructing contract's funds (`addr` in the stack representation above)
- `context_addr`: the address of the current execution context (e.g. what `ADDRESS` would put on the stack)

Gas Calculation:
- `gas_cost = 5000`: base cost
- `gas_refund = 24000`: base refund
- **If** `balance(context_addr) > 0 && is_empty(target_addr)` (sending funds to a previously empty address):
    - `gas_cost += 25000`


## AB: CALL\* Operations
Gas costs for `CALL`, `CALLCODE`, `DELEGATECALL`, and `STATICCALL` ops.
A big piece of the gas calculation for these operations is determining the gas to send along with the call.
There's a good chance that you are primarily interested in the `base_cost` and can ignore this additional calculation, because the `gas_sent_with_call` is consumed in the context of the called contract, and the unconsumed gas is returned.
If not, see the `gas_sent_with_call` [section](#ac-gas-to-send-with-call-operations).

Similar to selfdestruct, `CALL` incurs an additional cost if it forces an account to be added to the state trie by sending a nonzero amount of eth to an address that was previously empty.
"Empty", in this case is defined according to [EIP-161](https://eips.ethereum.org/EIPS/eip-161) (`balance == nonce == code == 0x`).

Terms:
- `call_value`: the value sent with the call (`val` in the stack representation above)
- `target_addr`: the recipient of the call (`addr` in the stack representation above)
- `mem_expansion_cost`: the cost of any memory expansion required (see [A2](#a2-memory-expansion))
- `gas_sent_with_call`: the gas ultimately sent with the call

### AB-1: CALL

Gas Calculation:
- `base_gas = 700 + mem_expansion_cost`
- **If** `call_value > 0` (sending value with call):
    - `base_gas += 9000`
    - **If** `is_empty(target_addr)` (forcing a new account to be created in the state trie):
        - `base_gas += 25000`

Calculate the `gas_sent_with_call` [below](#ac-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AB-2: CALLCODE

Gas Calculation:
- `base_gas = 700 + mem_expansion_cost`
- **If** `call_value > 0` (sending value with call):
    - `base_gas += 9000`

Calculate the `gas_sent_with_call` [below](#ac-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AB-3: DELEGATECALL

Gas Calculation:
- `base_gas = 700 + mem_expansion_cost`

Calculate the `gas_sent_with_call` [below](#ac-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`

### AB-4: STATICCALL

Gas Calculation:
- `base_gas = 700 + mem_expansion_cost`

Calculate the `gas_sent_with_call` [below](#ac-gas-to-send-with-call-operations).

And the final cost of the operation:
- `gas_cost = base_gas + gas_sent_with_call`


## AC: Gas to Send with CALL Operations
In addition to the base cost of executing the operation, `CALL`, `CALLCODE`, `DELEGATECALL`, and `STATICCALL` need to determine how much gas to send along with the call.
Much of the complexity here comes from a backward-compatible change made in [EIP-150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md).
Here's an overview of why this calculation is used:

EIP-150 increased the `base_cost` of the `CALL` op from 40 to 700 gas, but most contracts in use at the time were sending `available_gas - 40` with every call.
So, when `base_gas` increased, these contracts were suddenly trying to send more gas than they had left (`requested_gas > remaining_gas`).
To avoid breaking these contracts, if the `requested_gas` is more than `remaining_gas`, we send `all_but_one_64th` of `remaining_gas` instead of trying to send `requested_gas`, which would result in an `OUT_OF_GAS_ERROR`.

Terms:
- `base_gas`: the cost of the operation before taking into account the gas that should be sent along with the call.
See [AB](#ab-call-operations) for this calculation.
- `available_gas`: the gas remaining in the current execution context immediately before the execution of the op
- `remaining_gas`: the gas remaining after deducting `base_cost` of the op but before deducting `gas_sent_with_call`
- `requested_gas`: the gas requested to be sent with the call (`gas` in the stack representation above)
- `all_but_one_64th`: All but floor(1/64) of the remaining gas
- `gas_sent_with_call`: the gas ultimately sent with the call

Gas Calculation:
- `remaining_gas = available_gas - base_gas`
- `all_but_one_64th = remaining_gas - (remaining_gas // 64)`
- `gas_sent_with_call = min(requested_gas, all_but_one_64th)`

Any portion of `gas_sent_with_call` that is not used by the recipient of the call is refunded to the caller after the call returns. Also, if `call_value > 0`, a 2300 gas stipend is added to the amount of gas included in the call, but not to the cost of making the call.


## AF: INVALID
On execution of any invalid operation, whether the designated `INVALID` op or simply an undefined op, all remaining gas is consumed and the state is reverted to the point immediately prior to the beginning of the current execution context.
