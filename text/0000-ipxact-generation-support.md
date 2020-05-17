- 0000 IP-XACT Generation Support
- Start Date: 2019-01-18
- RFC PR: (leave this empty)
- Chisel-lang Issue: (leave this empty)

# Summary
[summary]: #summary

Provide a library that will enable generate **IP-XACT** XML collateral from an annotated **Chisel** circuit.
**IP-XACT** is defined in IEEE 1685-2014. 
It provides hardware descriptions that help in combining Hardware IP from multiple sources.

# Motivation
[motivation]: #motivation
A number of EDA vendors are able to ingest **IP-XACT** interface descriptions and use them to assist
in component integration and testing. Vendors have a number of IP standards such as AXI that have pre-existing tests
that can be used on a **Chisel** circuit that implements AXI if it can present **IP-XACT**.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

**chisel-lang**'s **IP-XACT** support comprises circuit, interface, and parameter annotations and **Firrtl** Stages and
transformation that can generate and **IP-XACT** XML file, that describes the following

- Interface Port Definitions
- Bus Definitions with mapping between logical and physical ports
- Address map information for where interfaces sit on a bus
- Parameter information that describes how the circuit was generated.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

**IP-XACT** support will be require dsptools and rocket chip facilities.
It will initially support **AXI4** and a small number of pre-existing devices within that scope.
The xml generation will be created from an annotation sequence and an underlying **Firrtl** file.

### Annotating a module that generates IP-XACT XML
A module that should have XML generated will require that the developer mark the module for generation
with the `Ipxact(moduleInstance)` command. This will add two annotations, `IpxactModuleAnnotation` and 
`IpxactBundleMemoizerTrigger` that reference `IpxactGeneratorTransform` and `IpxactBundleCaptureTransform` respectively.

### Memo-izing Bundles

Bundles that are referenced as IP-XACT annotations need both the top level bundle name as well as all the
lower level fields. To capture and retain the top level names there is second HighForm transform named
`IpxactBundleCaptureTransform` that will identify and preserve in `IpxactBundleName` annotations the original bundle 
names before they are expanded during FIRRTL lowering passes.

### Generation of **IP-XACT** will be triggered by a set of the following annotations:
  - **IpxactModuleAnnotation**
    - This targets a specific module.
    - Each annotation of this type will cause the creation of a separate IP-XACT .xml file.
  - **RegFieldDescMappingAnnotation**
  - **ParamsAnnotation**, with the following paramsClassName
    - freechips.rocketchip.amba.axi4.AXI4BundleParameters
    - ipxact.<parameter-block-name> for example: ipxact.GCDParams
    > Complexity: There can be multiple annotations initially whose `Target` components point to different bundles,
     for example, consider that there are two `data_in` and `data_out`.
    These annotations will be exploded into multiple annotations one per bundle field (recursive) so there will be
     `data_in_field_0`, ..., `data_in_field_N` and `data_out_field_0`, ..., `data_out_field_N`.
     This is good because I need to describe each of these.
     But, IP-XACT output wants these fields to be nested beneath there original bundle names.
     The expansion of the bundles into their fields obscures the original Bundle names.
     If I had two transforms one running before and one running after the expansion I could memoize the Bundle names.
     But there maybe more elegant solutions.
     
  - **AddressMapAnnotation**
  
### The Sections of the file will be 
  - **busInterfaces**
    - one or more **busInterface** describing the portmaps for for a bus
  - **addressSpaces**
    - more information required to construct this
  - **memoryMaps**
    - one or more **memoryMap** describing the individual fields of a **busInterface**
  - **model**
    - top level information for this file
  - **fileSets**
    - lists of files associated with this content
  - **parameters**
    - parameters used by **Chisel** to generate the module and devices
  
Most of the generation of the above will be explicitly based on annotations and the **Firrtl**.
Some elements, such as the busInterface portMaps will come from hand-coded mappings between ports and established **AXI4** names.
An initial reference example is in this [example in dsptools](https://github.com/ucb-bar/dsptools/blob/d56eaee509d60f14a32d7a7d85952e9494d2da34/rocket/src/test/scala/ipxact/AXI4GCD.scala).


**TODO**

- Additional patterns that will be supported
- Describe the annotations that will be supported
- File management that describes where generated IP will be placed. 


# Drawbacks
[drawbacks]: #drawbacks

**IP-XACT** has differing degrees of support amongst vendors and multiple versions of the standard can be found in the wild.
There may be a fair amount of hand editing in order to make the **IP-XACT** consumable by a particular vendor and that
work may not be transferable.
XML can be an awkward format with high overhead.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Providing a library for **IP-XACT** generation will simplify the work required of developers
- It will help ease barriers to adoption of **chisel-lang** ecosystem.
- It may provide tools for easier integration of 3rd party IP into **chisel-lang** projects.
- There no real alternative standards in this space.

# Prior art
[prior-art]: #prior-art

Several existing **Chisel** designs have generated by a small collection of project specific helpers added to the 
DspTools and RocketDspTools repos. 
This code needs to be standardized and moved to make it more generally accessible and applicable.
Note the occurrences of the search term ipxact on github in freechipsproject, ucb-bar, and ucb-art projects.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What is a reasonable set of annotations to implement this project
- Most of the pre-existing work relies on rocket-chip, is that too high a bar in order to generate **IP-XACT**

# Future possibilities
[future-possibilities]: #future-possibilities

- **IP-XACT** could be a useful tool in testing. Unit tests and circuit verification could be based on the types **IP-XACT** describes.
- **IP-XACT** could be ingested and used to build **chisel-lang** black boxes to aid 3rd party IP integration.