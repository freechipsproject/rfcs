- Feature Name: VHDL FIRRTL Emitter
- Start Date: 2019-07-15
- RFC PR:
- Chisel-lang Issue:

# Summary
[summary]: #summary

Extend FIRRTL to include a VHDL emitter in the same way that FIRRTL has a Verilog emitter.

# Motivation
[motivation]: #motivation

The motivation here is threefold:

1. **Ease of Interoperation with VHDL Technical and Cultural Environments:** Certain companies are entirely VHDL based. This comes up in certain industries (DoD vendors/contractors) and in certain regions (Europe tends to use VHDL). The lack of a VHDL emitter means that this forces potential Chisel users to deal with hardware generated in a language that they may have limited experience understanding. Similarly, VHDL design shops wanting to use Chisel must then migrate to mixed-language flows which may deviate from what they currently use.

2. **Generators Should be Back-end Agnostic:** Chisel is a circuit generator. It should be able to generate the types of circuits that people want. There's a long push to increasing the number of supported circuit concepts, e.g., Asynchronous Reset. However, we should also be agnostic in terms of generated circuit language. Other languages already support multiple backends, e.g., [SpinalHDL](https://github.com/SpinalHDL/SpinalHDL) supports Verilog or VHDL emission.

3. **Demonstration of the Hardware Compiler Framework:** Chisel and FIRRTL, post Stage/Phase, are architected as an N-stage compiler *framework*. Chisel is a front-end, the FIRRTL compiler is a middle-end optimizing compiler, and FIRRTL's Verilog emitter is a back-end. Towards demonstrating (and dog-fooding) this compiler framework nature, it would be useful to build an additional backend (here, VHDL) so that we can better demonstrate and evaluate software architectural decisions. I.e., we don't have experimental evidence adding a back-end and this exercise may help hone the Stage/Phase work.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In the FIRRTL compiler, there are multiple different `Emitter`s that you can choose from. These are implemented as subclasses of `firrtl.Transform`, i.e., mathematical transformations on a `firrtl.CircuitState` (`(a: CircuitState) => CircuitState`). Concretely, each emitter transform reads in a FIRRTL circuit, does some pre-processing on it, and generates a `firrtl.EmittedAnnotation` which contains the emitted circuit as a `String`. The underlying types used in this process are shown below:

```scala
sealed abstract class EmittedComponent {
  def name: String
  def value: String
  def outputSuffix: String
}

sealed trait EmittedAnnotation[T <: EmittedComponent] extends NoTargetAnnotation {
  val value: T
}
```

An emitter transform does not write-back the emitted circuit to disk (there is no IO side-effect). Instead, the final `firrtl.options.Phase` of the FIRRTL compiler is a `firrtl.stage.phases.WriteEmitted` phase that looks for any `EmittedAnnotation`s and dumps them to the location specified by a `firrtl.options.TargetDirAnnotation`.

FIRRTL provides the following emitters:

- `ChiselEmitter`
- `HighFirrtlEmitter`
- `MiddleFirrtlEmitter`
- `LowFirrtlEmitter`
- `MinimumVerilogEmitter`
- `VerilogEmitter`
- `SystemVerilogEmitter`
- `MinimumVHDLEmitter`
- `VHDLEmitter`

All of these emitters are implemented using the approach above. They are `Transform`s that generated one or more `EmittedAnnotation`s which will be written back when the `WriteEmitted` phase runs. Some emitters are pass-through and simply take the existing in-memory circuit representation and convert it to a `String`. These include the `{Chisel, High, Middle, Low}` emitters. The Verilog and VHDL emitter families do Verilog or VHDL-specific pre-processing and generate `EmittedAnnotation`s holding strings of Verilog or VHDL. "Minimal" variants do the bare-minimum of required pre-processing, but do not do additional, optional optimizations (e.g., constant propagation, primitive operator grouping, etc.).

A user can specify which emitter (or emitters) they want using an explicit `RunFirrtlTransformAnnotation` requesting that a specific emitter run or they can use the FIRRTL fat jar as following:

```bash
# To emit a single file containing the whole design in VHDL
firrtl -E vhdl

# To emit one-file-per-module of Verilog
firrtl -e verilog

# To emit both Verilog and VHDL
firrtl -E verilog -E vhdl
```

Similarly, a user can use explicit annotations (a `Seq[Annotation]` or `AnnotationSeq`) to control which emitters run. Users, both existing and new should be able to use the simple `Array[String]` or `AnnotationSeq` APIs to control which emitter runs without having to deal with any internals.

This is a pure API addition to add the VHDL emitter and is not expected to change anything with other emitters. Any VHDL-specific optimizations will be relegated to only running inside the VHDL emitter transform.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Like the Verilog emitters, VHDL (or other) emitters should be entirely self-contained. This is necessary if optimizations, which may be divergent from downstream Phase/Transform needs, are required. In short, the VHDL emitter should be implemented just like the Verilog emitter in terms of architecture.

The VHDL emitter should be able to be run with other emitters without impact. E.g., the Verilog emitter and the VHDL emitter should be able to be run together.

# Drawbacks
[drawbacks]: #drawbacks


## Verilog/VHDL Fungibility

Verilog and VHDL are entirely fungible from an FPGA/ASIC toolflow perspective. Mixed-language (using both Verilog and VHDL) designs are supported by all major tool vendors. Consequently, for most intents and purposes, Verilog is just as good as VHDL (and vice versa) from the perspective of putting something on an FPGA or taping out a chip.

This contrasts with how things work for backends of a software compiler. A software compiler backend is targeting a different ISA and expected to be different. In this case, the VHDL backend can be viewed as being redundant whereas software compiler backends are orthogonal.

The fact that the Verilog and VHDL backends are fungible/redundant then has problems of maintainability and validation. For maintainability, adding the VHDL backend means that any features going into either backend needs to be available in the other. This puts more of a load on existing FIRRTL developers and will slow down our cadence. For validation, you should be able to tape-out using either the VHDL or Verilog path and get the same result. This means that an implementation of this RFC would require doing some kind of equivalence checking on the Verilog and VHDL output. Doing this with open source tools may not be possible.

## Open Source Tooling

VHDL has no open source compiler equivalent to [Verilator](https://github.com/verilator/verilator). Some open source tools do exist, notably [GHDL](https://github.com/ghdl/ghdl), but the performance is not up to snuff with Verilator or VCS. Integration with GHDL would likely be needed as part of this PR and that adds further complexity of needing to maintain additional back-ends in testers. The existing factoring of things where post-emission back-ends live in testers may be worth reconsidering.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There's no major alternative design choice for how to implement a VHDL back-end and I think there's consensus there.

If we don't do this, then we have potential problems with adoption by VHDL shops and put another hurdle that potential Chisel users have to deal with.

## No VHDL Backend

However, the big alternative is to not do this and push everyone onto the Verilog backend. We could try to make this more palatable by adding examples that show integration of Chisel-generated VHDL with Verilog and/or emitting VHDL blackboxes that align with generated Verilog.

## Stage-based Backend

A potentially cleaner implementation would be to treat the VHDL backend as a separate stage. This can still be a `Transform`, but it would be wrapped up in a separate VHDL emitter `Phase` that could be run independently. This, while deviating somewhat from the current Verilog emitter family, would potentially be easier to maintain long-term and act as a demonstration for how to refactor the Verilog emitter going forward.

Nonetheless, it is likely easier to get all the VHDL emitter `Transform`s integrate with the way things work now and then refactor the emitter to be a separate stage.

# Prior art
[prior-art]: #prior-art

For prior art, the existing Verilog emitter is likely the best example of how to do this.

SpinalHDL seems to do just fine with maintaining two emitters, both Verilog and [spinal.core.internals.PhaseVhdl](https://github.com/SpinalHDL/SpinalHDL/blob/dev/core/src/main/scala/spinal/core/internals/PhaseVhdl.scala). SpinalHDL also seems to do a better job of giving users what they want and has some seeming traction in Europe where VHDL is more prevalent. From a "give the users what they want" perspective, there's good reason to add a VHDL back-end.

Additionally, there is a fair amount of similarity related to the old Chisel2 implementation of both a C++ emitter and a Verilog emitter. The initial motivation for this was that Verilator, at the time, was not as fast as it is now and Berkeley could do better with their own implementation. As Verilator improved, it made no sense to maintain an inferior backend that introduced questions of equivalence. A VHDL back-end is very similar in that it's expected to be less performant in simulation by open source tools (not it's fault, but a problem with missing tools) and has equivalence difficulties with Verilog.

Anecdotally, maintenance of the C++ and Verilog backends in Chisel2 was a known headache and moving to a single emitter in Chisel3 was a good approach.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should this be implemented as a separate `Stage` or should we follow the model of the existing Verilog emitter family?
- Is there enough motivation to add a VHDL back-end and do users actually want this?
- What resources do we have in place to maintain this and how can we go about verifying that both the Verilog and VHDL back-ends are equivalent?

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC is the first instance of adding a *new* back-end to the FIRRTL compiler. This motivates two future directions:

- This should be a model for how new back-ends should be added.
- This should be a starting point for thinking about what back-end `Stage`s should look like. This would either be part of this RFC as it evolves or it would be part of a future RFC about building back-end stages.

It would be beneficial if we had a dedicated owner of the VHDL back-end and if we had dedicated VHDL users. Without either of these, a VHDL back-end is likely to diverge and become difficult to maintain. Granted, we can't have VHDL users now if we don't emit VHDL. Any users would be mixed-language ones.
