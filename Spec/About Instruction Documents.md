# About Instruction Documents

Within the `Spec/` folder that contains this very document you are reading, there is the subdirectory `Spec/Instructions`. It contains the actual instruction specifications along with opcodes, encoding format, exception machinery, flags & prefix logic and more.

Note on the Exceptions section you'll see per each instruction document; those only list **specialized** exceptions that the instruction may raise on its own. It will not list all the generic ISA rule based exceptions. Those are valid with every instruction and each one is documented in [this document](./Interrupt%20&%20Exceptions.md#general-rchitectural-exceptions).  
If any blacklist condition is valid, the processor raises `Nonsensical Instruction Exception (NIE)`.  
If operand requirements are not met, the processor raises `Nonsensical Instruction Exception (NIE)`.  

It is also advised that you view [this document](./Opcode%20Map%20Overview.md) first in order to get a grab on how the opcode map is organized.
