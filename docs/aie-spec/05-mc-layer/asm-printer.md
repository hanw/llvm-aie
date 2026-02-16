# AIE MC Assembly Emission

Scope analyzed:
- `llvm/lib/Target/AIE/InstPrinter/*`
- `llvm/lib/Target/AIE/MCTargetDesc/AIETargetAsmStreamer.*`
- `llvm/lib/Target/AIE/MCTargetDesc/AIETargetELFStreamer.*`

## 1. Assembly syntax and printer architecture

Inst-printer stack:
- AIE1: `AIEInstPrinter` (`AIEGenAsmWriter.inc`)
- AIE2: `AIE2InstPrinter` (`AIE2GenAsmWriter.inc`)
- AIE2P: `AIE2PInstPrinter` (`AIE2PGenAsmWriter.inc`)
- Shared operand/bundle behavior for AIE2/AIE2P: `AIECommonInstPrinter`

`printInst` path:
- If instruction is composite (operands are nested `MCInst`), print each sub-instruction in sequence.
- Sub-instructions are separated by `";\t"`.
- If non-composite, print alias if enabled, otherwise canonical instruction mnemonic.

Alias handling:
- Controlled by `-aie-no-aliases` (`NoAliases` option).

## 2. Bundle/VLIW notation in textual assembly

Composite packet printing is flat and semicolon-separated, not brace-wrapped.

Observed behavior (`AIECommonInstPrinter::printInstr` and AIE1 variant):
- each slot instruction printed as a standalone mnemonic,
- bundle members joined as:
  - `inst_slot0;\tinst_slot1;\tinst_slot2 ...`

This matches parser-side bundling, where `;` is the end-of-bundle delimiter token.

## 3. Operand printing rules

From `AIECommonInstPrinter`:
- Registers: emitted by target register name table (`getRegisterName`).
- Immediates:
  - AIE1/AIE2: prefixed with `#`
  - AIE2P: no automatic `#` prefix in common printer path
- Expressions (`MCExpr`): printed directly; symbol refs on AIE1/AIE2 also get `#` prefix.

From `AIEInstPrinter` (AIE1-specific implementation):
- Immediates always printed as `#<value>`.
- `printImmOffset<k>` emits adjusted immediate (`imm + k`) for certain accumulator-high alias operands.

## 4. Directive handling through target streamers

`AIETargetAsmStreamer` prints option directives as raw text:
- `.option push`
- `.option pop`
- `.option rvc`
- `.option norvc`
- `.option relax`
- `.option norelax`

`AIETargetELFStreamer` provides no-op implementations for the same calls (object output path), so these are textual-assembly concerns only.

## 5. Public/Binary-facing implications

The InstPrinter defines canonical textual rendering that tools/tests rely on:
- semicolon-separated VLIW bundle syntax,
- alias-vs-canonical policy,
- immediate sigil policy by target triple.

The streamer layer defines which target directives are emitted in `.s` output and how those directives are ignored/consumed in ELF emission mode.

## 6. Noted caveat

Parser and printer immediate-prefix behavior should be validated specifically for AIE2P:
- parser base logic expects immediate tokens beginning with `#`,
- common AIE2P printer path does not automatically add `#` for immediates.

This may be handled by generated asm-writer operand formatting, but it is not explicit in handwritten code.
