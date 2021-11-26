# Research


During my PhD, I worked on [CPC](#cpc), an experimental dialect of the C
language, designed to write concurrent programs. A CPC program is compiled
into plain C through a series of source-to-source transformations.

During my Post-Doc, I worked on a formal model of a user-mode fragment of the
[POWER instruction set](#power), for the [REMS
project](https://www.cl.cam.ac.uk/~pes20/rems/) in Cambridge.

You can find find my publications below, or on
[my ResearchGate profile](https://www.researchgate.net/profile/Gabriel-Kerneis).

## Continuation-Passing C (CPC) <a id="cpc"></a>

CPC was a programming language for writing concurrent systems,
designed and developped by [Juliusz Chroboczek](irif.fr/~jch) and Gabriel Kerneis.

The CPC programmer manipulates very lightweight threads, choosing whether they
should be cooperatively or preemptively scheduled at any given point; the CPC
program is then processed by the CPC translator, which produces highly
efficient event-loop code.  This approach gives the best of both worlds: the
relative convenience of programming with threads, and the low memory usage of
event-loop code.

The semantics of CPC is defined as a source-to-source translation from CPC into
plain C using a technique known as *conversion into Continuation Passing
Style*.

CPC has been used to write Hekate, a BitTorrent seeder designed to handle
millions of simultaneous torrents and tens of thousands of simultaneously
connected peers, and to compile coroutines in the <a
href="http://www.qemu.org">QEMU emulator</a> during <a
href="https://www.google-melange.com/archive/gsoc/2013">Google Summer of
Code</a> 2013.

### Code

**CPC, Hekate and QEMU/CPC are not maintained anymore**, but their archived source code is still available:

* [CPC source code](github.com/kerneis/cpc) 
* [Hekate source code](github.com/kerneis/hekate) 
* [QEMU/CPC source code](github.com/kerneis/qemu) 

### Publications

Workshop paper on Hekate:

> Gabriel Kerneis, Juliusz Chroboczek.
> [CPC: programming with a massive number of lightweight threads](cpc-places11.pdf).
> PLACES'11 (2011). [Slides](cpc-places11-slides.pdf).

Workshop paper on QEMU/CPC:

> Gabriel Kerneis, Charlie Shepherd, Stefan Hajnoczi.
> [QEMU/CPC: Static Analysis and CPS Conversion for Safe, Portable, and Efficient Coroutines](qemu-cpc.pdf).
> PEPM'14, San Diego, CA, USA, January 20-21, 2014.

Journal article on CPC compilation:

> Gabriel Kerneis, Juliusz Chroboczek.
> [Continuation-Passing C, compiling threads to events through continuations](cpc-2012.pdf).
> Higher-Order and Symbolic Computation 24(3): 239-279 (2011).
> [Companion paper](TR-Kerneis-Chroboczeck-2012.pdf).

PhD thesis:

> Gabriel Kerneis.
> [Continuation-Passing C: Program Transformation for Compiling Concurrency in an Imperative Language](kerneis-phd-thesis.pdf).
> PhD thesis. Laboratoire PPS, Université Paris Diderot (2012).

Experiment on an alternative version of CPC where local variables are saved in
environments rather than lambda-lifted:

> Matthieu Boutier, Gabriel Kerneis.
> [Generating events with style](kerneis-boutier-2013.pdf).
>  Unpublished draft (2012).

Report on some early benchmarks:

> Gabriel Kerneis, Juliusz Chroboczek. 
> [Are events fast?](are-events-fast.pdf).
>  Technical report, Université Paris Diderot (2009).


## POWER ISA modelling <a id="power"></a>

### Code

* [The Sail ISA specification language](https://github.com/rems-project/sail/).
* I worked more specifically on the
  [Sail IBM POWER ISA model, automatically generated from IBM XML documentation](https://github.com/rems-project/sail/tree/sail2/old/power).

### Publication

> Kathryn E. Gray, Gabriel Kerneis, Dominic Mulligan, Christopher Pulte, Susmit Sarkar, and Peter Sewell. 
> [An integrated concurrency and core-ISA architectural envelope definition, and test oracle, for IBM POWER multiprocessors](micro-48-2015.pdf).
> MICRO-48, Waikiki, Hawaii, USA, December 5-9, 2015.

