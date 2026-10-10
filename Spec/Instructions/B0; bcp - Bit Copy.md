## Opcode(s) 10-21; Instruction `bcp - Bit Copy`

Copies bit N of an operand into the PSW (Processor Status Word) CF (Carry Flag) bit.

### Instruction Encoding

| Opcode | Instruction | O/E | TSZ |      Description     |
| :----: | :---------: | :-: | :-: | :------------------: |
|   B0   | `bcp v, n4` | OTO |  N  | Loads bit n4 of v.   |

### Instruction Operand Encoding

| O/E | Operand #1 | Operand #2 |
| :-: | :--------: | :--------: |
| OTO | OTO DB:DST | OTO DB:SRC |

### Flags Affected

The values of `ZF`, `OF` and `SF` are preserved.

`CF`: Set to the relevant bit.

### Specialized Exceptions

#### Nonsensical Instruction Exception (NIE)

If the bit N is an unsigned value above `15` for `TSZ = N`, or above `7` for `TSZ = 8`.

### Blacklists

For `O/E = OTO`:
- Three-Operand Escape

Prefixes (all opcodes):
-  F  (Preserve Status Flags)
- SOP (Source Port Override)

### Pseudocode

```
dst := src;
```
