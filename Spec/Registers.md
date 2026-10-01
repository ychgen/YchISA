# Registers

The Ych16 ISA features 7 general-purpose registers, each of word-size (16-bits). Registers have strong roles and purposes therefore violating those roles usually impose limitations, causing more instructions just moving things around. Whereas by leaning into these roles, you'll be able to make use of addressing capabilities the best and also enjoy usually shorter instruction encodings along with less instructions. For ordinary arithmetic though, using any register works just fine as they are all perfectly capable of being directly addressed. But the accumulator register is usually the best choice as a primary accumulation target because it has certain special encodings.

| Register Abb. | Register Name | Direct-Addressable | Base/Indirect-Addressable | Index-Addressable |
| :-----------: | :-----------: | :----------------: | :-----------------------: | :---------------: |
|   `RA`,`ra`   | Accumulator   |        Yes         |            Yes            |        Yes        |
|   `RB`,`rb`   | Base          |        Yes         |            Yes            |        No         |
|   `RC`,`rc`   | Counter       |        Yes         |            No             |        Yes        |
|   `RI`,`ri`   | Index         |        Yes         |            No             |        Yes        |
|   `RO`,`ro`   | Ordinal       |        Yes         |            No             |        Yes        |
|   `RP`,`rp`   | Pointer       |        Yes         |            Yes            |        No         |
|   `RS`,`rs`   | Stack Pointer |        Yes         |            Yes            |        No         |

The `RS` register always points at the top of the stack, at least that's what every stack instruction and architectural rules presume. The Ych stack grows downward, i.e. enlarging the stack causes the stack pointer to decrease, shrinking the stack causes the stack pointer to increase. The stack push operation order is 1. decrement pointer, 2. store element. The reverse is true for a pop operation.

When any of these 7 registers are encoded as general-purpose registers, they each have a 3 bit ID for the encoding. The encoding follows alphabetical order, and the table below will represent it:

| Register | Encoding |
| :------: | :------: |
|   `ra`   |   `000`  |
|   `rb`   |   `001`  |
|   `rc`   |   `010`  |
|   `ri`   |   `011`  |
|   `ro`   |   `100`  |
|   `rp`   |   `101`  |
|   `rs`   |   `110`  |

The ID `111` is usually reserved for something else, and points no legal **word-sized general-purpose register**. For instance, this specific register ID encoding within a **Unit Operand Descriptor** encodes the addressing mode for that operand as **Shorthand Displacement**, not that of any particular register.

When encoding one of the base-addressable registers, this 2-bit format is used instead:

| Register | Encoding |
| :------: | :------: |
|   `ra`   |   `00`   |
|   `rb`   |   `01`   |
|   `rp`   |   `10`   |
|   `rs`   |   `11`   |

When encoding one of the index-addressable registers, this 2-bit format is used instead:

| Register | Encoding |
| :------: | :------: |
|   `ra`   |   `00`   |
|   `rc`   |   `01`   |
|   `ri`   |   `10`   |
|   `ro`   |   `11`   |

## Byte Registers

The registers RA, RC, RI and RO are the 4 GPRs that can be byte-addressed directly, as in both their low and high byte of the whole word, when transaction size is 8-bits. Committing to a byte register will only update the part that was being referred to, i.e. writing to the high byte will keep the low byte intact.

Encoding table for byte registers is as follows:

| Register | Encoding |
| :------: | :------: |
|   `la`   |   `000`  |
|   `lc`   |   `001`  |
|   `li`   |   `010`  |
|   `lo`   |   `011`  |
|   `ha`   |   `100`  |
|   `hc`   |   `101`  |
|   `hi`   |   `110`  |
|   `ho`   |   `111`  |

`l*` denotes the low byte of the register `r*` whereas `h*` denotes the high byte of the register `r*`.  
You could alternatively think of the encoding as the MSB denoting `low/high` and the lower 2 bits encoding the identity of the 4 registers.
