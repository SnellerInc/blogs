# Building a SQL VM in AVX-512 Assembly

As part of open sourcing Sneller (https://github.com/SnellerInc/sneller), 
we would like to highlight its central innovations, the "interpreter,"
which is a bytecode-based virtual machine written almost entirely in AVX-512 assembly.
We are far from the first project to incorporate SIMD acceleration into
a query engine, but our interpreter is unusual
in that it is implemented *entirely* in assembly
despite it operating on flexibly-typed, row-oriented
data instead of strictly-typed columnar data.

## Why?

Our query engine falls roughly in to the same family
of SQL query engines as Presto, Impala and Spark
in the sense that we use horizontal scaling and hardware acceleration
to reduce query latency rather than relying on secondary indices.
Consequently, the performance of the query engine
*is* simply the performance of the interpreter;
any improvements we make to interpreter performance
translate directly into lower query latency or fewer
CPU cores necessary to achieve comparable latency.

A uniformly fast interpreter also provides *consistent*
performance across a large range of queries and use-cases,
which means developers don't have to spend nearly as much
time optimizing queries. The most important factor in predicting
how many resources a query will consume is the number of table bytes
scanned, not the phrasing of the SQL query.

Finally, a fast interpreter makes it possible to scale
queries *down* to a reasonable level of resource consumption
when the size of the input data is not terribly large.
If your interpreter is slow, your query engine will end up getting
out-performed by simple shell scripts until your data becomes
"big" enough to warrant a distributed query engine. (See 
["Command Line Tools can be 235x Faster than your Hadoop Cluster"](
https://adamdrake.com/command-line-tools-can-be-235x-faster-than-your-hadoop-cluster.html) for a motivating example).
Our design makes the best possible use
of the available hardware resources, regardless of whether
that is just a single CPU core or several hundred CPU cores
spread over a half dozen machines.

## Top-Down View

Queries pass through several different intermediate
representations before they reach a representation
that is actually executable by our bytecode VM.
Broadly speaking, there are two sub-problems involved
in executing a SQL query:

 1. Computing the best arrangement of "physical operators"
 (filtering, extended projection, aggregation, etc.) given the
 input query, the topology of the available compute resources,
 and all available sparse indexing metadata
 2. Executing each "physical operator" as efficiently as possible

The interpreter only concerns itself with the latter responsibility;
our query planner is responsible for the former.
By the time queries reach our interpreter, they have been decomposed into
"physical operators" that need to evaluate one or more SQL expressions
and pass the results to a subsequent operator.

For example, a query like

```SQL
SELECT SUM(y)
FROM my_table
WHERE x < 3
```

would evaluate `x < 3` for each input row
and discard any rows where that expression doesn't
evaluate to `TRUE`. Then it would accumulate a running
total of the sum of `y`.

Textually, we could describe the pipeline of query operations as
```
ITERATE my_table -> FILTER (x < 3) -> AGGREGATE SUM(y) -> output
```

Internally, our query engine uses the [Amazon Ion](https://amzn.github.io/ion-docs/docs/binary.html) binary format.
Each row of data is an ion structure composed of zero of more fields
which themselves may be structures or lists (much like JSON records).
Evaluating `x < 3` means locating the `x` field in
each structure, unboxing it as a number, and then comparing it
against the constant `3`.
(We don't typically know in advance that `x` will be a number,
or even that `x` will be present at all, but we'll see that the
interpreter deals with data polymorphism just fine.)

### Expression AST

The `WHERE x < 3` clause of a SQL query will
eventually end up in the `vm.Filter` physical operator.
Here's what `go doc` has to say about creating a `vm.Filter`:

```
package vm // import "."

func NewFilter(e expr.Node, rest QuerySink) *Filter
    NewFilter constructs a Filter from a boolean expression. The returned Filter
    will write rows for which e evaluates to TRUE to rest.
```

The `expr.Node` type is the interface satisfied by
any AST expression. In our `x < 3` example, the concrete
value passed as `e` would be:

```go
&expr.Comparison{
   Op:    expr.Less,
   Left:  &expr.Path{First: "x"},
   Right: expr.Integer(3),
}
```

### SSA IR

The first stage of expression compilation is to convert the input AST into a
[Single Static Assignment](https://www.cs.cornell.edu/courses/cs6120/2022sp/lesson/6/)-based
intermediate representation that we can more easily optimize.
Our SSA instructions generally map 1-to-1 to bytecode VM instructions.

Our SSA representation doesn't use basic blocks,
as they would complicate the intermediate representation,
and they would make it harder to convert the SSA IR into
something that can be vectorized. Instead, we use "predicated execution,"
which means that we pass a one-bit predicate to each operation
in order to indicate whether or not its execution should be short-circuited
into a no-op. As we will see later on, using predicated execution instead
of ordinary control-flow makes it much easier to implement the final
bytecode program using vector instructions.

Our `x < 3` example above would be decomposed
into SSA that looks something like the following:
```
b0.k0 = init                   # initial row base and lane mask
v1.k1 = findsym b0 k0 $x       # compute x pointer and mask
f2.k2 = tofloat v1 k1          # unbox x as a float and mask
   k3 = cmplt.imm.f f2 k2 $3.0 # compute result mask from comparison
ret: k3
```

(The real SSA produced for this example is slightly more
complex in practice, as it handles integer-encoded `x` values as well.)

The SSA value `init` simply describes the initial state of the VM,
with `b0` representing the row "base pointer" and `k0` representing
the initial predicate value (i.e. `TRUE`). The `init` SSA operation
doesn't generate any code; it exists only so that other instructions
can actually reference the "initial state" (registers, etc.) of the VM
as it is entered.

The `findsym` operation takes a struct base (`b0`), a symbol (in this case `x`),
and a predicate (`k0`) and produces a "fat pointer" for the associated struct field
(`v1`) along with a new predicate (`k1`) indicating whether the field is actually
present.

The `tofloat` operation takes the `v1` fat pointer and unboxes it into a floating-point number `f2` as long as `k1` is set. The returned predicate `k2`
is only set when `k1` is set *and* the value in `v1` was successfully interpreted
as a floating-point number.

Finally, the `cmplt.imm.f` operation produces our return value by evaluating
`f2 < $3.0` when `k2` is set. The value `k3` is our return value from this program;
if `k3` is set, then the expression `x < 3` has evaluated to `TRUE` and the row
can be passed to subsequent physical operators.

You will notice that we don't evaluate `x < 3` as `TRUE` when `x` is
not present or not a number. If `x` is not present, then `k1` will be unset
and thus `k2` and `k3` (the return value) will be as well.
Similarly, if `x` is a string, then `k2` will be unset, and thus so will`k3`.
(Since we are only interested in expressions that evaluate to `TRUE` in this example,
we don't have to concern ourselves with the details of whether or not
the expression evaluated to `MISSING` or `NULL` or `FALSE`; there are other
situations where we have to be a bit more careful about how to interpret predicate
values.)

### Bytecode

Once we have built the SSA representation of our AST
expression, we have to convert it into a representation that is actually
executable by our bytecode VM.

The bytecode VM just executes a linear sequence of instructions,
so our first order of business is computing a valid execution
ordering of the SSA instructions. A post-order traversal of
the SSA program beginning at the return instruction will always
produce a valid instruction ordering, and in our example
there is only one possible post-order traversal of instructions,
so the result is trivial. (Often there is more than one possible
instruction ordering, so we've developed some heuristics for
picking a "good" ordering that minimizes the number of stack
shuffling instructions we have to produce.)

Once we have determined the instruction ordering, we need to
produce virtual stack operations to ensure values are in the correct registers at the
beginning of each bytecode operation. The bytecode VM uses a number
of virtual registers (called `B`, `V`, `K`, `H`, and `S`), and typically
these registers correspond to one or two physical registers in the assembly code.
For example, the virtual `K` register always holds the predicate argument to
an instruction, and on Intel processors this corresponds to the physical `k1`
register. Similarly, the `V` register always points to the first boxed argument
to a bytecode operation, and on Intel this corresponds to `zmm30` and `zmm31`,
where `zmm30` holds sixteen rows worth of 32-bit offsets relative to
`rsi` and `zmm31` holds the corresponding lengths.
During the final pass of bytecode compilation, we allocate
some temporary storage for holding virtual register values that need
to be spilled and restored across instructions.

Every bytecode is encoded as a two-byte instruction number,
followed by zero or more bytes of immediate data that are specific
to the instruction. For example, the `findsym` operation needs to accept
a symbol ID to search for, so its encoding includes a 4-byte immediate value
that the assembly code loads and compares against. Similarly, `cmp.imm.f`
needs to accept an 8-byte float64 immediate, so the complete instruction is
ten bytes long.

It's worth noting that our *model* of the virtual machine doesn't know
or care about vectors of rows; it only models the set of high-level operations
that have to occur for each row, and it uses predicated execution instead
of control flow in order to make it *easier* to vectorize the implementation.
Porting the VM to a different architecture would mean picking a
reasonable mapping between VM registers and hardware registers and reimplementing
each bytecode instruction according to that register convention.
Nothing at the SSA or bytecode level would change if the opcode implementations
were operating on one row, two rows, or a thousand rows at once.
(The AVX-512 implementation operates on groups of sixteen rows at a time.)

#### Brief Interlude: AVX-512 and Mask Registers

For us, the most important feature of AVX-512 as compared to
AVX2 is the presence of "mask" (or "predicate") registers.
Most AVX-512 instructions accept a mask register (`k0` through `k7`)
operand that causes the lanes corresponding to the zero bits of the
mask to remain untouched by the instruction in question.

For example, here's a masked 32-bit vector addition (`vpaddd`):
```
vpaddd %zmm12,%zmm7,%zmm7{%k5}
```

The instruction above adds the sixteen 32-bit integers in `zmm12`
to the corresponding 32-bit integers in `zmm7` and writes the results
into `zmm7`, but *only for the lanes where k5 is set*. In other words,
this instruction does not modify any lanes in `zmm7` where `k5` is unset.

Per-lane predication is important to us because it allows us
to emulate diverging control between lanes without using branches.
For example, imagine we need to implement the following in
AVX-512 assembly:
```C
if (x < 3) {
    x += 2;
} else {
    x -= 1;
}
```

Instead of using control flow, we can simply compute a mask
for `x < 3` and then use that mask and its complement for each
of the two arms of the `if/else` statement:
```asm
;; assume we have broadcast
;;  1 into %zmm1,
;;  2 into %zmm2,
;;  3 into %zmm3
vpcmpltd %zmm3, %zmm0, %k1        ;; k1 = zmm0 < zmm3, per lane
knotw    %k1, %k2                 ;; k2 = ~k1
vpaddd   %zmm0, %zmm2, %zmm0{%k1} ;; zmm0 += 2 iff k1
vpsubd   %zmm1, %zmm0, %zmm0{%k2} ;; zmm0 -= 1 iff k2
```

It is technically possible to emulate predicated execution using
AVX2 features, but it is considerably more cumbersome than in AVX-512.

Fortunately for us, ARM SVE/SVE2 and the RISC-V Vector Extension
both implement predicated execution, so we are not necessarily stuck
with only AVX-512 as an implementation target.
(Once AWS c7g instances are available we will be able to
make an attempt at an SVE port.)

### Assembly Routines

Many of the bytecode assembly routines consist of only a few machine instructions.
For example, `cmp.imm.f` corresponds to the `bccmpltimmf` bytecode operation,
which is defined in the source code with a few macros:
```
TEXT bccmpltimmf(SB), NOSPLIT|NOFRAME, $0
  BC_CMP_OP_F64_IMM($VCMP_IMM_LT_OQ)
  NEXT_ADVANCE(8)
```

If you disassemble this code from a compiled binary,
you end up with the following instruction sequence:
```
000000000065ed80 <bccmpltimmf>:
  65ed80:	62 f2 fd 48 19 20    	vbroadcastsd (%rax),%zmm4
  65ed86:	c4 e3 f9 30 d1 08    	kshiftrw $0x8,%k1,%k2
  65ed8c:	62 f1 ed 49 c2 cc 11 	vcmplt_oqpd %zmm4,%zmm2,%k1{%k1}
  65ed93:	62 f1 e5 4a c2 d4 11 	vcmplt_oqpd %zmm4,%zmm3,%k2{%k2}
  65ed9a:	c5 ed 4b c9          	kunpckbw %k1,%k2,%k1
  65ed9e:	48 0f b7 58 08       	movzwq 0x8(%rax),%rbx
  65eda3:	48 83 c0 0a          	add    $0xa,%rax
  65eda7:	48 81 e3 ff 00 00 00 	and    $0xff,%rbx
  65edae:	48 8d 15 eb 9c 25 00 	lea    0x259ceb(%rip),%rdx        # 8b8aa0 <opaddrs>
  65edb5:	ff 24 da             	jmp    *(%rdx,%rbx,8)
```

The input `f2` floats are passed in `%zmm2` and `%zmm3` as
sixteen doubles (eight in `%zmm2` and eight in `%zmm3`), and the
input make is provided in `%k1`. The first few instructions
simply broadcast the immediate from the instruction stream (`3.0` in this case)
into `%zmm4` and synthesize a new `%k1` value based on the input mask
and the result of the two eight-lane comparison operations.

The final five instructions of the operation perform dynamic
dispatch of the next bytecode instruction.
In pseudo-code, the assembly is evaluating:

```C
next = *(rax+8) & 0xff; /* load next instruction */
rax += 10;              /* adjust %rax to point after next insn */
goto *opaddrs[next];    /* jump! */
```

Folks who have worked on interpreters will recognize
this as a classic "indirect threaded" interpreter:
we compute the address of the next bytecode instruction
by indirecting through a table of addresses.
In this case, we're using the hardware register `rax`
as the "virtual program counter" that is adjusted as we execute.

The indirect branch at the end of each bytecode routine
is much more predictable than you might imagine,
as the number of branch targets *per instruction* for any particular sequence
of bytecode operations tends to be close to 1.
(If a bytecode program is ten virtual instructions, but the ten instructions
are *distinct from one another*, then the indirect branch that occurs
at the end of each virtual instruction is perfectly predictable; it always branches
to the following instruction's implementation!) In practice the "dispatch overhead"
of our interpreter is so low that it is barely measurable;
the fact that we are doing sixteen lanes worth of work in each
bytecode instruction helps amortize the cost of dynamic dispatch
over lots of "real" work.

### Trampolines

Once we have an executable sequence of bytecode operations,
we need a way of entering the VM from portable Go code.
Each of the physical operators implements a "trampoline"
function (written in assembly) that populates the right
VM registers with the initial state for a group of up to
sixteen rows, invokes the bytecode by jumping into the first virtual instruction,
and then takes the return value from the VM and does something
sensible with it. In the case of `vm.Filter`, the `evalfilterbc`
trampoline uses the final `%k1` value returned from the VM to
compress away input row delimiters using `vpcompressd`, and then
the remaining rows are passed to a subsequent physical operator.

Trampoline routines can typically accept an arbitrarily large
number of input rows. (We usually aim to process several hundred
rows of data per call.) Importantly, this means that we spend
basically zero time in actual portable Go code once we have
compiled the bytecode; the "inner loop" of the VM is implemented
entirely in assembly. *This is a critical piece of the design,
because it means the Go language does not meaningfully constrain
the performance of the VM as compared to "faster" alternatives
like C/C++ or Rust.*

### Further Reading

Our `WHERE x < 3` example covers a vanishingly small portion
of the capabilities of our VM. As of this writing we have more
than 250 bytecode operations, spanning everything from hash trie
lookup to string matching to great-circle distance calculation.
There is also a considerable amount of complexity lurking in our
query planner in order to translate more advanced SQL queries
into something that fits into our relatively simple VM model.
We are planning on covering some of those topics in much more detail in future blog posts.

### Check it out

If you want to see more for yourself, please head over to the 
main repository at https://github.com/SnellerInc/sneller.
