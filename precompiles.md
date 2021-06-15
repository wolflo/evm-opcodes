# EVM Precompiles

References:
These files are useful, but should use the actual geth repo tho

- https://github.com/ethereum-optimism/optimism/blob/6d38f4731c449c3de1b0fc00d4ea18196073bd00/l2geth/params/protocol_params.go#L91
- https://github.com/ethereum-optimism/optimism/blob/6d38f4731c449c3de1b0fc00d4ea18196073bd00/l2geth/core/vm/contracts.go#L357

All numbers are for Istanbul:

| Address | Name           |   Gas   | Notes |
| :-----: | :------------- | :-----: | :---- |
|  0x01   | ecrecover      |  3000   |       |
|  0x02   | sha256hash     | Dynamic |       |
|  0x03   | ripemd160hash  | Dynamic |       |
|  0x04   | dataCopy       | Dynamic |       |
|  0x05   | bigModExp      | Dynamic |       |
|  0x06   | bn256Add       |   150   |       |
|  0x07   | bn256ScalarMul |  6000   |       |
|  0x08   | bn256Pairing   | Dynamic |       |
|  0x09   | blake2F        | Dynamic |       |


