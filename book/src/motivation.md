# Motivation

At our company, we generally use C/C++ for bare-metal programming and Rust for software that runs on a full operating system. There are several reasons why moving everything to Rust could benefit us:

1. **The general "Why Rust?"**: memory safety, modern syntax, pattern matching, a growing community, and more.
2. **A single language**: when there is one language instead of two, there is less for the average developer to learn and keep up with.
3. **No more duplicated code**: logic shared between our Rust and C codebases is currently written twice, because the options for calling between Rust and C (FFI, the foreign function interface) are not good enough. These FFI options are examined later in this research.
4. **Something better than C**: switching gives us a chance to move to something better than the C we currently use.
