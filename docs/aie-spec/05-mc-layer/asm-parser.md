# AIE MC Assembly Parsing

Scope analyzed:
- `llvm/lib/Target/AIE/AsmParser/AIEBaseAsmParser.h`
- `llvm/lib/Target/AIE/AsmParser/AIEBaseOperand.*`
- `llvm/lib/Target/AIE/AsmParser/AIE1AsmParser.cpp`
- `llvm/lib/Target/AIE/AsmParser/AIE2AsmParser.cpp`
- `llvm/lib/Target/AIE/AsmParser/AIE2PAsmParser.cpp`

## 1. Parser architecture

AIE uses a shared template parser (`AIEBaseAsmParser<Parser, BundleType, OperandType>`) with per-arch overrides for:
- register name matching (`matchRegister`),
- identifier parsing (`parseIdentifier`),
- match/emit diagnostics (`matchAndEmitInstruction`),
- post-match semantic checks (`validateInstruction`).

Operand representation is `AIEBaseOperand` with 3 kinds:
- token,
- register,
- immediate (`MCExpr`).

## 2. Instruction and operand parsing

Core flow (`parseInstruction`):
1. push mnemonic token as operand 0,
2. parse first operand if present,
3. parse comma-separated remaining operands,
4. do not consume end-of-statement (used by bundle logic).

Operand forms:
- Register/identifier
- Immediate: `#<integer|identifier|(...)>`
- Address forms via `parseIndirectOrIndexedMode`:
  - `[ptr]`
  - `[ptr, reg]`
  - `[ptr, #imm]`

Immediate expression variants:
- symbol refs,
- integer expressions (including negative),
- parenthesized expression wrapped as `AIEMCExpr::VK_AIE_GLOBAL`.

Call symbol operands are wrapped as `AIEMCExpr::VK_AIE_CALL` (`parseCallSymbol`).

## 3. Bundle parsing and delimiters

Bundle behavior is integrated into `processMatchedInstruction` and `emitBundle`:
- instructions are accumulated into an `AIE::MCBundle`.
- if `Bundle.canAdd(Inst)` fails, parser reports `"incorrect bundle"` and clears bundle state.
- end-of-bundle is detected when token is not `;`.
- at end-of-bundle, parser emits a composite packet instruction.
- missing slots are auto-filled with slot-specific NOP opcodes (`Formats.getSlotInfo(Slot)->getNOPOpcode()`).

So textual `;` is both:
- separator between instructions,
- explicit bundle continuation marker.

## 4. Per-architecture custom parse behavior

### AIE1
- `validateInstruction` currently mostly pass-through.
- mnemonic spellcheck hook via `AIEMnemonicSpellCheck` on invalid mnemonic.

### AIE2
- immediate constraints enforced for selected instructions in `validateInstruction`:
  - PADDA/PADDS, PADDB, MOVA, stack-pointer specific ranges/alignment.
- `.3d` instruction-name convention updates `d0..d3` to `d*_3d` register variants when mnemonic ends with `.3d`.
- mnemonic spellcheck via `AIE2MnemonicSpellCheck`.

### AIE2P
- similar range/alignment checks for PADDA, MOVA in `validateInstruction`.
- `.3d` register conversion equivalent to AIE2.
- overrides `parseImmediate` to preserve `#` token explicitly before base parse.
- mnemonic spellcheck via `AIE2PMnemonicSpellCheck`.

## 5. Directive parsing status

`AIEBaseAsmParser::ParseDirective` is currently a stub implementation (returns `true` directly).

Implication:
- handwritten directive parsing is incomplete in this class,
- target streamer directive output exists, but parser-side custom directive handling is not implemented here.

## 6. Binary interface implications

The parser defines accepted textual ABI for assemblable input:
- semicolon-driven VLIW packet assembly,
- bracketed addressing-mode syntax,
- required immediate sigil and expression forms,
- call/global expression tagging (`AIEMCExpr` variants) that drives downstream fixup/reloc behavior.
