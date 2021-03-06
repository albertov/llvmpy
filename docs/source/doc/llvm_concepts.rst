********************
LLVM Concepts 
********************

This section explains a few concepts related to LLVM, not specific to
llvmpy.

.. toctree::
   :hidden:

   
   

Intermediate Representation
===========================

The intermediate representation, or IR for short, is an in-memory data
structure that represents executable code. The IR data structures allow
for creation of types, constants, functions, function arguments,
instructions, global variables and so on. For example, to create a
function *sum* that takes two integers and returns their sum, we need to
follow these steps:

-  create an integer type *ti* of required bitwidth
-  create a function type *tf* which takes two *ti* -s and returns
   another *ti*
-  create a function of type *tf* named *sum*
-  add a *basic block* to the function
-  using a helper object called an *instruction builder*, add two
   instructions into the basic block: 
    - an instruction to add the two
      arguments and store the result into a temporary variable 
    - a return
      instruction to return the value of the temporary variable

(A basic block is a block of instructions.)

LLVM has it's own instruction set; the instructions used above (*add*
and *ret*) are from this set. The LLVM instructions are at a higher
level than the usual assembly language; for example there are
instructions related to variable argument handling, exception handling,
and garbage collection. These allow high-level languages to be
represented cleanly in the IR.


SSA Form and PHI Nodes
======================

All LLVM instructions are represented in the *Static Single Assignment*
(SSA) form. Essentially, this means that any variable can be assigned to
only once. Such a representation facilitates better optimization, among
other benefits.

A consequence of single assignment are PHI (Φ) nodes. These are required
when a variable can be assigned a different value based on the path of
control flow. For example, the value of *b* at the end of execution of
the snippet below:

.. code-block:: c

   a = 1;
   if (v < 10)
       a = 2;
   b = a; 

cannot be determined statically. The value of '2' cannot be assigned to
the 'original' *a*, since *a* can be assigned to only once. There are
two *a* 's in there, and the last assignment has to choose between which
version to pick. This is accomplished by adding a PHI node:

.. code-block:: c

   a1 = 1;
   if (v < 10)
       a2 = 2;
   b = PHI(a1, a2); 

The PHI node selects *a1* or *a2*, depending on where the control
reached the PHI node. The argument *a1* of the PHI node is associated
with the block *"a1 = 1;"* and *a2* with the block *"a2 = 2;"*.

PHI nodes have to be explicitly created in the LLVM IR. Accordingly the
LLVM instruction set has an instruction called *phi*.


LLVM Assembly Language
======================

The LLVM IR can be represented offline in two formats

-  a textual, human-readable form, similar to assembly language, called
   the LLVM assembly language (files with .ll extension)
-  a binary form, called the LLVM bitcode (files with .bc extension)

All three formats (the in-memory IR, the LLVM assembly language and the
LLVM bitcode) represent the *same* information. Each format can be
converted into the other two formats (using LLVM APIs).

The `LLVM demo page <http://www.llvm.org/demo/>`_ lets you type in C or
C++ code, converts it into LLVM IR and outputs the IR as LLVM assembly
language code.

Just to get a feel of the LLVM assembly language, here's a function in
C, and the corresponding LLVM assembly (as generated by the demo page):

.. code-block:: c

   /* compute sum of 1..n */
   unsigned sum(unsigned n) {
       if (n == 0)
           return 0;
       else
           return n + sum(n-1);
   } 

The corresponding LLVM assembly:

.. code-block:: llvm

   ; ModuleID = '/tmp/webcompile/_7149_0.bc'
   target datalayout = "e-p:64:64:64-i1:8:8-i8:8:8-i16:16:16-i32:32:32-i64:64:64-f32:32:32-f64:64:64-v64:64:64-v128:128:128-a0:0:64-s0:64:64-f80:128:128-n8:16:32:64"
   target triple = "x86_64-linux-gnu"

   define i32 @sum(i32 %n) nounwind readnone { 
   entry:
      %0 = icmp eq i32 %n, 0          ; [#uses=1]
      br i1 %0, label %bb2, label %bb1

   bb1:     ; preds = %entry
      %1 = add i32 %n, -1          ; [#uses=2]
      %2 = icmp eq i32 %1, 0     ; [#uses=1]
      br i1 %2, label %sum.exit, label %bb1.i

   bb1.i:     ; preds = %bb1
      %3 = add i32 %n, -2          ; [#uses=1]
      %4 = tail call i32 @sum(i32 %3) nounwind   ; [#uses=1]
      %5 = add i32 %4, %1          ; [#uses=1]
      br label %sum.exit

   sum.exit:     ; preds = %bb1.i, %bb1
      %6 = phi i32 [ %5, %bb1.i ], [ 0, %bb1 ]         ; [#uses=1]
      %7 = add i32 %6, %n                                    ; [#uses=1]
      ret i32 %7

   bb2:       ; preds = %entry
      ret i32 0
   }

Note the usage of SSA form. The long string called ``target datalayout``
is a specification of the platform ABI (like endianness, sizes of types,
alignment etc.).

The `LLVM Language Reference <http://www.llvm.org/docs/LangRef.html>`_
defines the LLVM assembly language including the entire instruction set.


Modules
=======

`Modules <./llvm.core.Module.html>`_, in the LLVM IR, are similar to a
single *C* language source file (.c file). A module contains:

-  functions (declarations and definitions)
-  global variables and constants
-  global type aliases for structures

Modules are top-level containers; all executable code representation is
contained within modules. Modules may be combined (linked) together to
give a bigger resultant module. During this process LLVM attempts to
reconcile the references between the combined modules.


Optimization and Passes
=======================

LLVM provides quite a few optimization algorithms that work on the IR.
These algorithms are organized as *passes*. Each pass does something
specific, like combining redundant instructions. Passes need not always
optimize the IR, it can also do other operations like inserting
instrumentation code, or analyzing the IR (the result of which can be
used by passes that do optimizations) or even printing call graphs.

This LLVM `documentation page <http://www.llvm.org/docs/Passes.html>`_
describes all the available passes, and what they do.

LLVM does not automatically choose to run any passes, anytime. Passes
have to be explicitly selected and run on each module. This gives you
the flexibility to choose transformations and optimizations that are
most suitable for the code in the module.

There is an LLVM binary called
`opt <http://www.llvm.org/cmds/opt.html>`_, which lets you run passes on
bitcode files from the command line. You can write your own passes (in
C/C++, as a shared library). This can be loaded and executed by +opt+.
(Although llvmpy does not allow you to write your own passes, it does
allow you to navigate the entire IR at any stage, and perform any
transforms on it as you like.)

A "pass manager" is responsible for loading passes, selecting the
correct objects to run them on (for example, a pass may work only on
functions, individually) and actually runs them. ``opt`` is a
command-line wrapper for the pass manager.

LLVM defines two kinds of pass managers:

-  The
   `FunctionPassManager <http://llvm.org/docs/doxygen/html/classllvm_1_1FunctionPassManager.html>`_
   manages function or basic-block passes. These lighter weight passes
   can be used immediately after each generated function to reduce
   memory footprint.

-  The
   `PassManager <http://llvm.org/docs/doxygen/html/classllvm_1_1PassManager.html>`_
   manages module passes for optimizing the entire module.


Bitcode
=======

LLVM IR can be represented as a bitcode format for disk storage. It is
`suitable for fast loading by JIT
compiler <http://llvm.org/docs/LangRef.html#introduction>`_. See `LLVM
documentation <http://llvm.org/docs/BitCodeFormat.html>`_ for detail
about the bitcode format.


Execution Engine, JIT and Interpreter
=====================================

The *execution engine* implements execution of LLVM IR through an
interpreter or a JIT dynamic compiler. An *execution engine* can contain
multiple modules.

    **Note**

    Inter-module reference is not possible. That is module ``A`` cannot
    call a function in module ``B``, directly.

