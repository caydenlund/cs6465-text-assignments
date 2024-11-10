# Class Project Selection: C vs. Rust

**Author: Cayden Lund (u1182408)**

Rust is often noted for its strong type system, memory safety guarantees, and modern concurrency features, yet it's generally thought to be slightly slower than C.
I'm interested in understanding the specific factors that contribute to this performance difference.
My passion is in compilers, and I love Rust as a language, so this project aligns very closely with my interests.
I feel that understanding the performance slowdowns is particularly relevant to modern operating systems; Linux and Windows have been integrating Rust code into their kernels, several new operating systems in Rust are quickly gaining popularity, and world governments are pressuring companies to transition to memory-safe languages in the next few years.
Therefore, having a good understanding of the performance challenges that Rust is facing, and especially their causes, is a relevant and important issue.

I plan to first analyze key differences between C and Rust in their memory models, concurrency techniques, and low-level optimizations.
I am particularly interested in examining pointer aliasing guarantees that Rust provides.
Because Rust programs strictly manage ownership, the compiler should be able to understand that, for example, two arguments to a particular function can't point to the same mutable region of memory, and that therefore loop unrolling can safely be done without producing incorrect results.

Then, I want to identify performance bottlenecks by profiling programs written in each language and examining the assembly instructions.
I'll run benchmarks with different access patterns, different types of mutability, and different loads to see how the behaviors differ under different conditions.

I want to experiment with introducing unsafe code into Rust programs to see whether I can optimize the speed using techniques that wouldn't be possible with pure safe Rust, and I will identify what can be done without breaking the overall memory safety of the algorithm.

Finally, I'll take what I've learned to try to identify key techniques for writing Rust programs that optimize the runtime and still guarantee memory safety.
