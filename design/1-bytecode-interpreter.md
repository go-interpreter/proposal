# Proposal: Design of a bytecode interpreter for Go

Author: Sebastien Binet

Last updated: 2016-08-26

Discussion at https://github.com/go-interpreter/proposal/issue/1.

## Abstract

We propose to design and implement a bytecode interpreter for Go,
which will be the foundation for a Go [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop).

## Background

It is common in science or exploratory work to iterate on a piece of code
to solve a given problem.
Having an interactive conversation with your program, _via_ an interactive
prompt (aka a REPL), greatly speeds up such exploratory work: one can easily
iterate on various algorithms, modifying the state of your program and data,
and write new types and functions to _e.g._ plot the new state of your data.

A side benefit of such an interpreter is the ability to embed it inside
a Go application and provide both scriptability and extensibility.
Designing such an API is outside the perimeter of this proposal.

There are currently already partial solutions or whole implementations
of a Go REPL on the market but none of those meets the following requirements:

- easy `go get` installation
- implement the whole Go language
- be a real REPL, not just an "on-the-fly re-compilation + re-run the whole snippet" approach
- JIT-able
- performant

## Proposal

We propose to break the complicated issue of bringing a complete interpreter
for Go (interactivity, whole-program interpretation, runtime, native functions,
external functions, JITing, parsing source code, ...) into small pieces.

The current proposal only deals with describing the bytecode interpreter
(its overall design and its components), the opcodes and instructions which
can be found in a bytecode stream and how these bytecodes can be interpreted and
acted upon by the interpreter.

There are many ways to implement an interpreter and as many options
for the interpretation process:

1. directly interpret from the source code
2. interpret the source code after it has been transformed into an AST
3. compile statements into bytecode instructions that are then executed

We propose to go with option 3).
Option 1) doesn't lend itself to optimizations nor very efficient execution.
Option 2) is somewhat better: there are ways to programmatically manipulate
and transform an AST.
But with option 3) we should be able to reuse the whole corpus of optimizations
coming from the new SSA backend of the official `gc` Go compiler.
As explained in Rob Pike's talk at GopherCon-2016: ["The Design of the Go Assembler"](https://talks.golang.org/2016/asm.slide),
the `cmd/internal/obj` package can be seen as a rather portable assembly language.
This paves the way for considering it as a portable intermediate representation
(IR) of Go code.

The proposal is thus to use this conduit as the general infrastructure to
generate the opcodes and bytecode for the new Go VM.
The concrete _modus_ _operandi_ for leveraging `cmd/internal/obj` and
the whole `gc` compiler infrastructure might still need to be properly fleshed
out, but here are the current options:

- create a proper `GOARCH` architecture directly under `cmd/internal` like
  the other `GOARCH=amd64`, `GOARCH=s390x`, etc... architectures and aim for
  Go 1.8, (we would need to declare our plans [here](https://groups.google.com/forum/#!topic/golang-dev/098vr4999Tk))
- vendor `cmd/compiler` at a given Go version (_e.g._ 1.7) and work off it,
  aiming for integration at a later date (if at all possible),
- ???

### Instructions, opcodes and bytecode format

We propose to reuse the opcodes and bytecode format as described in the [Dis VM](http://www.vitanuova.com/inferno/papers/dis.pdf)
specification paper.
The `Dis` VM was able to execute [Limbo](https://en.wikipedia.org/wiki/Limbo_%28programming_language%29)
code.
`Limbo` and `Go` share a common lineage and present similar features
(channels, `select`, garbage collector, packages) so many (if not all) of
the opcodes our VM will need are already present and the instruction set has
been formally described.
The on-disk object file format and overall organization has also been specified
in the above paper.

We intend to follow the general spirit of the specifications of the `Dis` VM
and condense it inside a package named `dice`.
The implementation of `dice` should be done from first principles,
without looking at the `Dis` source code
This is to ensure that `dice` can be licensed under `BSD-3`.

The various `opcode`s are listed here:

```
00 nop    20 headb  40 mulw  60 blew    80 shrl
01 alt    21 headw  41 mulf  61 bgtw    81 bnel
02 nbalt  22 headp  42 divb  62 bgew    82 bltl
03 goto   23 headf  43 divw  63 beqf    83 blel
04 call   24 headm  44 divf  64 bnef    84 bgtl
05 frame  25 headmp 45 modw  65 bltf    85 bgel
06 spawn  26 tail   46 modb  66 blef    86 beql
07 runt   27 lea    47 andb  67 bgtf    87 cvtlf
08 load   28 indx   48 andw  68 bgef    88 cvtfl
09 mcall  29 movp   49 orb   69 beqc    89 cvtlw
0A mspawn 2A movm   4A orw   6A bnec    8A cvtwl
0B mframe 2B movmp  4B xorb  6B bltc    8B cvtlc
0C ret    2C movb   4C xorw  6C blec    8C cvtcl
0D jmp    2D movw   4D shlb  6D bgtc    8D headl
0E case   2E movf   4E shlw  6E bgec    8E consl
0F exit   2F cvtbw  4F shrb  6F slicea  8F newcl
10 new    30 cvtwb  50 shrw  70 slicela 90 casec
11 newa   31 cvtfw  51 insc  71 slicec  91 indl
12 newcb  32 cvtwf  52 indc  72 indw    92 movpc
13 newcw  33 cvtca  53 addc  73 indf    93 tcmp
14 newcf  34 cvtac  54 lenc  74 indb    94 mnewz
15 newcp  35 cvtwc  55 lena  75 negf    95 cvtrf
16 newcm  36 cvtcw  56 lenl  76 movl    96 cvtfr
17 newcmp 37 cvtfc  57 beqb  77 addl    97 cvtws
18 send   38 cvtcf  58 bneb  78 subl    98 cvtsw
19 recv   39 addb   59 bltb  79 divl    99 lsrw
1A consb  3A addw   5A bleb  7A modl    9A lsrl
1B consw  3B addf   5B bgtb  7B mull    9B eclr
1C consp  3C subb   5C bgeb  7C andl    9C newz
1D consf  3D subw   5D beqw  7D orl     9D newaz
1E consm  3E subf   5E bnew  7E xorl
1F consmp 3F mulb   5F bltw  7F shll
```

We reserve the right to rename some of these `opcode`s to better reflect
the naming conventions of our source language, Go.

### Virtual Machine

Once a Go package, command or code snippet has been compiled to our `dice` bytecode,
that bytecode needs to be somehow executed.
This job is performed by the `dice.VM` virtual machine:

```go
package dice

type VM struct {
	frame   *frame
	globals []reflect.Value
}

type frame struct {
	vm     *VM
	caller *frame
	locals []reflect.Value
	pc     int // program counter
	code   []instruction
}

type instruction struct {
	opcode byte
	amode  byte   // address mode
	addrs  uint64 // operands (src1, src2, dst)
}

func (vm *VM) run() {
	run(vm.frame)
}

func run(fr *frame) {
	for {
code:
		for _, code := range fr.code {
			switch exec(fr, code) {
				case cfReturn:
					return
				case cfNext:
					// fetching next instruction
				case cfJump:
					break code
			}
		}
	}
}

func exec(fr *frame, code instruction) cfKind {
	switch code.opcode {
		case opADDF:
			// dst = src1 + src2
			fr.pc++
		case opCALL:
			run(&frame{caller:fr, pc:0, code: from(src)})
		case opRET:
			// fetch result if any
			return cfReturn
		case opGO:
			go func() {
				run(&frame{caller:fr})
			}()
		// etc...
	}
}
```

At this moment, the proposal is to be able to byte compile this simple Go package:

```go
package main

func add(i, j int) int {
	return i+j
}

func main() {}
```

and in a later stage, be able to run `add(40, 2)`.

## Rationale

Why do we implement yet another Go interpreter and a REPL?
Aren't there already enough of them?

Here is a list of alternatives:

- [llgoi](https://github.com/llvm-mirror/llgo/blob/master/cmd/llgoi/llgoi.go) is a JIT-enabled interpreter built on top of `LLVM` and `llgo`.
  The first issue with `llgoi` is the somewhat painfull installation process.
  This pain point should be resorbed with time (and also by providing [snap based](https://groups.google.com/forum/#!msg/llgo-dev/ny8MgDlNkng/8kEvgzfuCQAJ)
  isntallations of `llgoi`.
  But the main issue is that `llgo` development is behind that of the reference
  implementation of `Go`: `gc`.
  Also, the pace of development of `LLVM` itself (very fast) and the version skew
  that may result on users' machines *might* set the scene for difficult user
  support and debugging sessions.

- [ssainterp](https://github.com/go-interpreter/ssainterp) and [ssadump -run](https://godoc.org/golang.org/x/tools/cmd/ssadump)
  are based on the SSA suite developped at [golang.org/x/tools/go/ssa](https://godoc.org/golang.org/x/tools/go/ssa).
  They are able to parse and interpret a vast majority of valid Go code,
  but lack an interactive interpreter mode.
  `ssadump` code is also clearly stated as *NOT* meant to be used as a
  production-grade interpreter for Go but merely as an adjunct for testing
  the SSA construction algorithm.

- [igo](https://github.com/sbinet/igo) and [go-eval](https://github.com/sbinet/go-eval)
  are projects salvaged from the pre `Go-1` era.
  `go-eval` does not lend itself easily to compilation optimizations and lacks
  support for `imports`, `goroutines`, type creation, ...

- [gore](https://github.com/motemen/gore) supports the whole Go language but
  does not (completely cleanly) preserve state or side effects between
  2 interactive commands: `gore` recompiles on-the-fly your Go snippets and
  re-executes them.

It seems necessary to implement some kind of a virtual machine to be able
to provide an efficient and truly interactive interpreter for Go.

The same question can be also raised about reimplementing a whole new VM.
Couldn't we have somehow reused an already existing VM?
`Python`, `Lua`, `JVM` and `Dis` come to mind.
`Dis` is LGPL and thus not easily integrable in the usual Go ecosystem.
`Python` and `Lua` have more permissive licenses, but their reference
implementation are written in `C`, bringing either performance issues on the
table (`cgo`) or throwing `go-get`-ability out of the window.
There are however `Go` implementations (partial or complete) of these VMs:

- https://github.com/Shopify/go-lua/blob/master/vm.go
- https://github.com/flowlo/gothon/blob/master/frame.go

The following issue at this point is the adequacy of their respective VM
instructions sets with the Go language.

Finally, why do we use the `Dis` VM instructions set, instead of a more recent
or more in vogue set, such as [LLVM bitcode](http://llvm.org/docs/BitCodeFormat.html)
and its associated [LLVM assembly](http://llvm.org/docs/LangRef.html), or the
nascent [`wasm` bytecode](https://webassembly.github.io/) format?

The `LLVM` solution suffers (to a lesser extent) from the same issues than the `llgoi` approach.
We should note though there exists a pure-Go project to interact with the `LLVM` `IR`:
[llir/llvm](https://github.com/llir/llvm).
This project is still a work in progress at this time of writing (August 2016).

`wasm` is probably a very strong and sensible option, and poised to take over
the whole web industry.
Unfortunately, there is only a work in progress `C/C++` project at the moment (August 2016),
so it is probably a bit early to write code to target it.
However, `wasm` is definitely a backend to monitor: `gopherjs`, a project transpiling
Go code into `JavaScript` will probably target it at some point.

## Compatibility - Open issues

There are a few interesting issues when interpreting Go code in an interactive
fashion.

1. Should we allow mid-way imports of packages ?
   ```
   go> slice := []string{"HELLO", "GO"}
   go> import "strings"
   go> println(strings.ToLower(slice[0]))
   ```

   What if `slice` was instead named `strings`?
   Should we allow shadowing of variables by package identifiers?
   Should we instead re-shadow the package identifier with the variable
   identifier?
   The latter seems like the more idiomatic Go behaviour, or at least the
   behaviour a gopher would expect if she were to write the program in
   a compiled environment (_i.e.:_ with `goimports` putting the `import`
   statement at the top)

2. Support for `cgo` and `import "C"` ?
3. Support for packages with assembly ? (from the `stdlib` or otherwise)
4. Calls to `syscalls` ? Should they be somehow recognized and performed
   on a dedicated `goroutine`? What should `os.Exit` do? and how?
5. How to efficiently implement iteration over maps?
6. How to implement `unsafe`? Should we?
7. How to implement the definition of new types?
   Package `reflect` has some support for this (`StructOf`, `ArrayOf`, ...) but
   it currently has no support for defining new interface types nor any new
   named types.
8. In an interactive interpreter, how do we define methods for a named type?
   When, and how, do we tell the interpreter that the method set of a given
   named type is done?
9. What is the most efficient way to write the `opcode` dispatch loop?
   A huge switch? ([go-lua](https://engineering.shopify.com/79963844-announcing-go-lua)
   reported issues with huge switches and migrated to a jump table.)

## Implementation

1. `dice.{VM,frame,instruction}` implementation leading to the execution
   of already decoded instructions,
2. implementation of the bytecode stream decoder,
3. implementation of the bytecode encoder,
4. implementation of the interactive prompt of the REPL (with limitations),
5. implementation of dynamically importing packages at the REPL level.
   This probably needs either a working `buildmode=plugin` from the `go` tool,
   or a complete handling of dynamically loading bytecode object files.

