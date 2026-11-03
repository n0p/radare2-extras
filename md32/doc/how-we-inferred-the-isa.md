# How did we infer the ISA?

The uSIP and SCQ16 tables of this plugin (`isa/usip_isa.json`,
`isa/scq16_isa.json`) are not vendor specifications. They were inferred from a
single modem firmware image by static analysis, without a chip to run the code
on. This note explains how, twice: first for a reader who knows what a
disassembler is but has never reverse-engineered an instruction set, then for
a reader who wants to argue about the validity of the inference. The numbers
are those of the uSIP table at the time the table was frozen; the SCQ16 and
MS15E30 cases are summarised at the end.

If you only read one sentence: every form in the table was admitted because an
independent oracle, measured over the whole corpus, said so, and the
confidence level it carries (`high`, `medium`, `low`, `format-only`, see
`doc/usip.md` §3) records how strong that oracle was.

## 1. The short version

### What the problem was

A modem firmware image is one binary blob. Inside it runs code for several
small DSP cores, and the main one, uSIP, has no public instruction manual. You
cannot run it on a PC, and this work had no live chip to poke at either. So the
question was: can you work out what every instruction means just by looking at
the bytes? That is static analysis in its purest form.

Think of it as learning a language from a huge pile of text with no
dictionary. You would fail, except that we had three cheat sheets:

* **A symbol table** shipped with the firmware. It gives the name, start and
  end of every function. Over five thousand functions.
* **Known source code** for some of them. The C library inside the modem is
  newlib, which is open source, and several speech codecs are public reference
  code. So for `strlen`, `malloc`, `free` and friends we know what the compiler
  was asked to do.
* **A second build of the same firmware** for a different device. Same code,
  compiled with slightly different addresses.

### How the analysis was done

**Step 1: build a corpus.** A loader cuts the blob into functions and the
functions into 16-bit units. Everything after this is measured over all of
it, not over hand-picked examples. That matters: a rule that works on ten
instructions is a guess, a rule that works on 780 thousand is a fact.

| Corpus | Count |
|---|---|
| Functions | 5,392 |
| 16-bit units | 1,393,838 |
| Instruction packets | 471,907 |

**Step 2: find the structure before the meaning.** This is a VLIW machine, so
several operations are packed together and execute in parallel. The first
discovery was that one bit in each 16-bit unit marks "this closes the packet".
How do you prove that without a manual? You use an oracle. An oracle is some
independent fact you can test against. Here the oracle was the function
table: branch targets must land at the start of a packet. Under the bit-15
rule they do about 98 percent of the time. Under random cutting, about 60
percent. That gap is the proof.

**Step 3: control flow first.** Calls and branches are the easiest
instructions to validate, because you can compute the target and check
whether it lands on a function start. Over ninety percent of the decoded
calls did. Once calls, jumps, conditional branches, returns and hardware
loops are nailed down, you have the skeleton of every function.

**Step 4: work out the packet layout.** Instructions turned out to be a head
unit plus zero, one or two tail units, and the tails are not necessarily next
to their head. The decoder enumerates the legal ways to pair heads and tails,
scores each reading, and keeps the cheapest one. It also learned that only
the first slot of a packet can be control flow, and that the rest of an open
packet is its delay slot.

**Step 5: differential analysis.** This is the cleverest trick. Align the two
firmware builds function by function. In most functions the only bytes that
differ are addresses, shifted by a known amount. So any bit field whose value
changes by exactly that shift must be an address field. That told us the
exact width of displacement fields, which forms are 48 bits long, and which
bits are sign bits, with no guessing.

**Step 6: C anchors for arithmetic.** For functions with known source, you
know for example that a getter returns `r0 == 0` or `r0 < 1`. The instruction
sitting at that spot is therefore a set-on-compare. Each anchor names one
encoding, and the encoding then names thousands of other sites.

**Step 7: residue rounds.** Decode everything, collect the units that still
have no name, cluster them by bit pattern, propose a reading, measure it
against the oracles, accept or reject. Repeat. The decoder went through
seventeen versions this way. Mnemonic coverage went from about four percent
at the start to just over ninety-seven percent.

**Step 8: be honest about confidence.** Every form in the table carries a
level: `high`, `medium`, `low` or `format-only`. A trailing question mark on a
mnemonic means the encoding is fixed but the operation is not. The decoder
never prints a confident name it cannot back with evidence.

**Step 9: freeze it into a tool.** The result is a JSON instruction table, a
generator that turns it into C headers, and this plugin, with test vectors so
regressions are caught.

### The limits

Static analysis has a ceiling. Some questions cannot be settled by bytes
alone: whether two operations in one packet see old or new register values,
which of two equally cheap readings is right, what a count instruction
returns on zero. For each of these the open question is recorded with the
exact bytes and the result each hypothesis predicts, so that one register
snapshot from real hardware decides it. See `doc/usip.md` §5 and
`doc/scq16.md` §5 for the lists.

The lesson: reverse engineering is not cleverness with a hex editor. It is
forming a hypothesis, finding an independent oracle, measuring over the whole
corpus, and writing down the confidence you actually earned.

## 2. The long version

### Problem formulation

The object is a firmware image for a MediaTek MT6877 modem containing code for
several heterogeneous DSP cores, with no ISA specification and no execution
environment. The task is instruction-set inference: recover, for one core, a
decoder that maps byte sequences to packets of operations with operands,
control-flow semantics and types, from a single static image. Formally this
is unsupervised grammar induction over a token stream with an unknown
tokenizer, where the only supervision is weak and indirect. Three weak
oracles were available: a function table shipped with the firmware giving
symbol boundaries, open-source provenance for a subset of functions (newlib
and public speech-codec reference code), and a second build of the same
firmware linked for a different device. Everything else had to be derived
from the corpus itself.

| Corpus | Count |
|---|---|
| Functions | 5,392 |
| 16-bit units | 1,393,838 |
| Packets | 471,907 |
| Final named coverage, units | 97.4 % |

### Method

**Structural identification by oracle agreement.** Each structural hypothesis
was expressed as a tokenization rule and scored by the fraction of
oracle-implied facts it makes true, against an explicit null. The packet
delimiter was accepted because branch targets fall on packet starts in about
98 percent of cases under the rule, against a 60 percent base rate for
arbitrary unit boundaries. Rules for packet alignment, control flow
restricted to slot zero, and the delay-slot model were accepted the same way.
Hypotheses that predicted equally well were not distinguished, and that is
recorded rather than resolved by taste. An earlier "variable-length bundles
with a last-unit marker" model was measured and rejected because targets
landed inside its bundles.

**Control flow as the anchor set.** Branch and call encodings are the only
forms whose correctness can be externally verified per instance, since a
decoded target either hits a function start or it does not. The validated
set is large enough to treat as ground truth for everything downstream. This
is deliberately the opposite of starting from the opcode space.

| Class | Instances | Target at boundary |
|---|---|---|
| 32-bit conditional branch | 30,324 | 99.6 % |
| Relative call | 16,713 | 99.2 % |
| Absolute call | 2,630 | 100 % |
| Hardware loop | 3,360 | 99.8 % |

**Differential analysis as a controlled experiment.** The second firmware
build is the same source under a different link map. After function-level
alignment, the units that differ are almost entirely address fields
displaced by a per-region constant. This turns field identification into a
measurement rather than a guess: a bit range is an address field with a
given width and sign convention if and only if its value changes by exactly
the region displacement. This fixed the 22-bit displacement of relative
calls, the 48-bit segmented immediate forms, the semantics of a rotated-sign
immediate, and exposed that DDR code links against a specific TCM copy. The
method has a clear failure mode, orphan units whose partner was mis-paired,
and those are counted and reported.

**Semantic identification from source anchors.** For functions with known C,
the expected data flow at a site constrains the operation of the instruction
found there. One anchor fixes one encoding; the corpus then propagates it.
The set-on-compare family was named from four getters in a public codec
whose constants appear in the preceding subtraction. These anchors are also
the regression suite: twenty functions are re-decoded and checked on every
change.

**Packet decoding as combinatorial search with an explicit cost.** An
instruction is a head plus up to two tails that need not be adjacent. The
decoder enumerates all group partitions and round-robin head-tail
assignments the table admits and selects by a lexicographic cost: unnamed
units, then instruction count, then structural violations, then target
evidence, then a documented list of tie-break penalties derived from
measured regularities such as write-after-write within a packet (`doc/usip.md`
§1). No learned prior enters the selection, so every decision is auditable
against the table. Residual ties go through a fixed cascade (`doc/usip.md`
§7) and the packets it cannot decide are flagged as ambiguous in `opex`
rather than silently resolved.

| Decoder state | Ambiguous packets |
|---|---|
| Early version | 7.0 % |
| Frozen version | 0.20 % |

**Iterated residue analysis with adversarial verification.** Each round
decoded the full corpus, clustered the unnamed residue by bit pattern,
proposed readings, measured them against the oracles, and froze the table.
Seventeen decoder generations ran this loop. From the sixth round on, each
round was followed by an independent verification pass that re-measured
claims from raw bytes, and a critic pass that attacked the weakest
inferences. Refuted claims were kept in the working notes together with the
refutation. Each form carries a confidence level with the admitting evidence
named, and provisional mnemonics carry a visible `?` marker.

**Semantics that static analysis can and cannot settle.** The parallel-read
question, whether a DSP operation in a packet reads the old or the new value
of a register written by an ALU operation in the same packet, is not
decidable per site. It was closed statistically by a liveness argument: under
the new-value reading, the previous writer of that register would be dead
code. In the disputed class the previous writer has no other reader in 63.5
percent of well-read pairs, against 3.0 percent in a negative control where
the new value is certain, and 43 to 65 percent in positive controls where the
old value is certain. The distribution matches the old-value controls. Eight
source anchors agree, and an explicit search for counterexamples under the
same filter found none that survive. That is the strongest form of closure
available without a register snapshot. The plugin emits no ESIL for uSIP; the
SCQ16 ESIL, whose packets have the same parallel model, reads the sources of
a packet before any of its writes.

### Validity and limits

The main threats are circularity and selection. Circularity is bounded
because the oracles are external to the decoder and the cost function
contains no fitted parameters. Selection is addressed by measuring over the
whole corpus with null baselines, and by counting what each rule fails to
explain. Coverage was reported in three metrics because they disagree, and
the strict one (every unit of the instruction named) is the lowest.

What remains open is listed with the experiment that would settle it.
Packet-order semantics, the residual tied layouts, a handful of doubtful loop
forms and a double-write idiom each have identified sites where one register
snapshot after the packet discriminates between hypotheses, with predictions
precomputed per hypothesis.

### The other encodings served by this plugin

* **MS15E30** is publicly documented and the plugin's table for it was
  verified against the vendor disassembler (`doc/research.md`). It also served
  as a sanity check of the inference pipeline: the three RAKE cores of the
  same firmware turned out to use MS15E30, and the public table matched over
  99 percent of their words.
* **SCQ16** is a byte-granular encoding with a scalar model close to MD32.
  The same method applied, with the length rule learned per first byte from
  the function-boundary oracle (`doc/scq16.md` §1). The static ceiling was
  reached lower: the scalar forms are near complete, the vector operations
  are classified but mostly not named. The public test vectors
  (`test/scq16_vectors.json`) are the plugin's predictions for the sites
  where a hardware observation would settle the open readings; the working
  notes behind them check that every pair of competing hypotheses yields a
  distinct observable, so a single observation per site decides.
