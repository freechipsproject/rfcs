- Feature Name: fix-memory-within-when
- Start Date: 2019-07-01
- RFC PR: 
- Chisel-lang Issue: 

# Summary
[summary]: #summary

When all references to a memory are within the scope of a when statement, Firrtl connects the memory's clock through a validIf using the when's condition.
This RFC is to encourage working to a fix for this problem. Existing work and thinking are recorded here.

To review the full details of the original issue motivating this RFC go to 
[/firrtl/issues/702][https://github.com/freechipsproject/firrtl/issues/702]

# Motivation
[motivation]: #motivation

The following Chisel circuit:
```scala
class Stack(val depth: Int) extends Module {
  val io = IO(new Bundle {
    val push    = Input(Bool())
    val pop     = Input(Bool())
    val en      = Input(Bool())
    val dataIn  = Input(UInt(32.W))
    val dataOut = Output(UInt(32.W))
  })

  val stack_mem = Mem(depth, UInt(32.W))
  val sp        = RegInit(0.U(log2Ceil(depth+1).W))
  val out       = RegInit(0.U(32.W))

  when (io.en) {
    when(io.push && (sp < depth.asUInt)) {
      stack_mem(sp) := io.dataIn
      sp := sp + 1.U
    } .elsewhen(io.pop && (sp > 0.U)) {
      sp := sp - 1.U
    }
    when (sp > 0.U) {
      out := stack_mem(sp - 1.U)
    }
  }

  io.dataOut := out
}
```
generates the following selected low Firrtl.
```
    node _T_29 = gt(sp, UInt<1>("h0")) @[Stack.scala 31:14:@31.6]
    node _GEN_7 = validif(_T_29, clock) @[Stack.scala 31:21:@32.6]
    node _GEN_16 = validif(io_en, _GEN_7) @[Stack.scala 24:16:@11.4]
    stack_mem._T_35.clk <= _GEN_16 @[:@3.2]
```
  * **Current behavior?**
`validIf` behavior is undefined when `io_en` is false. meaning it could randomly cycle the memory's clock.
  * **Expected behavior?**
`memory.clk` should be directly connected to clock or `mux`ed on `enable` to be `clock` or `UInt<1>("h0")`. 

* **What is the use case for changing the behavior?**
The existing behavior causes inconsistent results in the interpreter and could affect other simulations or synthesized code.

* **Impact**
  - [x] no functional change
  - [ ] API addition (no impact on existing code)
  - [ ] API modification
  - [ ] unknown

* **Development Phase**
  - [x] request
  - [x] proposal



--------------------------------------------------------
***Discussion Summary (Developer Edits)***
**Underlying Problem: how to translate from a Chirrtl memory to a Firrtl memory.**

Chisel's memory API is to separate the memory declaration from the port declaration:
```scala
val m = Mem(4, UInt(32.W))
when(io.en) {
  m(io.addr) := 7.U
}
```

However, FIRRTL has a single memory declaration which includes declaring all ports:
```
mem m :
  dataType -> UInt<32>
  depth -> 4
  writer -> _T_84
  readLatency -> 0
  writeLatency -> 1
m._T_84.en <= io.en
m._T_84.addr <= io.addr
m._T_84.data <= UInt(7)
m._T_84.clk <= clock
```

To make generating FIRRTL easier, we introduced CHIRRTL which also separates memory declarations from port declarations, making Chisel's generation easier:
```
cmem m : UInt<32>[4]
when io.en :
  write mport _T_84 = m[io.addr], clock
  _T_84 <= UInt(7)
```

So the problem lies in converting the CHIRRTL memory representation into FIRRTL's. Fundamentally, a `when` statement (in any other context) implies a mux. However, a `when` around a CHIRRTL `mport` actually implies an **enable on that port**. Thus, the question is how to specially handle the address, enable, and clock values specified at the port declaration (and when) in CHIRRTL, but at the memory declaration in FIRRTL.

Fortunately, the enable and address are easy. For the enable, we can set it to zero at the memory declaration, and set it to one at the port declaration. Normal expand whens + constprop will resolve this to be correct. For the address, we can invalidate it at the memory declaration, before we expand when's. This will generate a `validif` on the address value after expand when's, but this is still behaviorally correct and has no impact on area (don't *have* to generate a mux from it).
```
mem m :
  dataType -> UInt<32>
  depth -> 4
  writer -> _T_84
  readLatency -> 0
  writeLatency -> 1
m._T_84.en <= UInt(0)
m._T_84.addr is invalid
m._T_84.clk <= ???
when io.en :
  m._T_84.en <= UInt(1)
  m._T_84.addr <= io.addr
  m._T_84.data <= UInt(7)
```

The problem is what to do with that (stinking) port clock. The current API is to set it to invalid, but as @chick pointed out, that obviously isn't correct and a bug (unless we redefine its semantics). We have 4 options:

1. Set the port clock to invalid at the memory declaration, set it to the clock value at the port declaration, and redefine connections of Clock type to ignore when and last connect semantics.
2. Pull the port clock value up from the port declaration to the memory declaration.
3. Set the port clock to 0 at the memory declaration, set it to the clock value at the port declaration, and let normal when semantics hold. Then, in the VerilogEmitter, recognize the special case where a clock mux can be optimized away if the mux enable is the same as the data enable.
4. Support mport directly in FIRRTL, and get rid of Chirrtl (forever).

Discussion of 1:
Pro - Simple implementation (like ~2 loc changes)
Con - API decision about how clocks and last connect semantics work, which probably has negative implications for future attempts to support generic clock muxing. Can be non-intuitive to users.

Discussion of 2:
Pro - Somewhat simple implementation (~100 loc changes)
Con - If a local clock, declared after the memory declaration, is used by a port to the memory, then you get a really non-intuitive error. **I don't see an easy way of solving this.**

Discussion of 3:
Pro - Clean resolution of how clocks/whens interact, without loss of expressivity. Opens the door to future generic clock muxes.
Con - Not a pure transformation, in that it will change the behavior of multiple-cycle writes on memories.


Current status: Tried out 1, 2, and 3 already, but I'm dissatisfied. Trying out option 4.
  


Why are we doing this? What use cases does it support? What is the expected outcome?

# Commentary
[guide-level-explanation]: #guide-level-explanation

From @azidar

Ok, after trying option 1, it works pretty well with all previous problems. However, there is a second problem. If a new clock is declared between the memory declaration and the port declaration, it fails:

```
circuit Unit :
  module Unit :
    input clock : Clock
    smem ram : UInt<32>[128]
    node newClock = clock
    infer mport x = ram[UInt(2)], newClock
```

Notice this is independent of `when` interactions. The reason is because when we drag the clock assignment from the port declaration to the memory declaration, the `newClock` has not been declared yet. If the user swaps `smem ram : UInt<32>[128]` with ` node newClock = clock`, it works. This seems to be very unintuitive behavior for a user.... 

Ok, I'm convinced that Option 1 won't work, because with the new `withClock` Chisel3 construct, it is very easy to express this problem.

```scala
class MultiClockMem extends Module {
  val io = IO(new Bundle {
    val en      = Input(Bool())
    val other   = Input(Clock())
    val addr    = Input(UInt(2.W))
    val out = Output(UInt(32.W))
  })

  val mem = Mem(4, UInt(32.W))

  when (io.en) {
    mem(io.addr) := 7.U
    io.out := 0.U
  }.otherwise {
    val local = Wire(Clock())
    local := io.other
    withClock(local) {
      io.out := mem(io.addr)
    }
  }
}
```

This means we actually need to formally resolve how Clock types interact with when statements in FIRRTL (it's no longer simply a product of how Chisel interfaces with Chisel).

However, I don't like making Clocks ignore when statements either, it means you can't express Clock muxing with whens, which would exclude a future pass which could swap out generic clock muxes with a custom clock mux.

So... maybe a third option is in order: Set the default clock to be asClock(0.U). This would work in simulation, but would require more changes to ReplSeqMem and VerilogEmitter to recognize you are muxing the mem port's clock on the same enable as the data enable (and thus can optimize away the mux).

After discussions, decided we should do option 4: Add mport as a first class FIRRTL construct which can reference DefMemory's, and which get's eliminated before MiddleFIRRTL. Eliminate ChirrtlForm and Chirrtl.
# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


# Future possibilities
[future-possibilities]: #future-possibilities

There may be better solutions, it is hoped this RFC will encourage open thinking on this issue