# Why Rust? Bare-Metal Edition.

In this repo we are going to examine using Rust in bare-metal programming as an alternative to the more common C/CPP.
The result of this research would be an mdbook.

## Motivation

In our company we generally use C/CPP for bare-metal programming. and Rust when running on a complete OS. \
There are multiple reasons why switching everything to Rust would benefit us.
1. The general "Why Rust?" (memory safe, modern syntax, pattern matching, growing community, etc...)
2. Using a single language is less knowledge the average developer to maintain.
3. Common Rust/C code currently written twice as the Rust-C FFI options are not that good (which will also be examined in this research).
4. Exploring the opportunity for something better than currently used C.

## Notes

1. The research addresses both C/CPP. but as most of the code is C we will most of the time just write C. when specifically talking about CPP features we will explicitly state it.
2. Consider the target audience might not really know embedded and bare-metal, so when addressing concepts we will explain them first and language specific details later.
3. As much as I care about technical advanced concepts, this research first priority is to address all the reasons for and against switching to Rust in the company. including issues from the motivation part of this file. this whole should be a chapter in the research.     

## PHASE 1 - START HERE

Create the main page, single opening sentance. \
differ resources and notes from the actual product. \
gitignore the mdbook build output (`book/book/`).

## PHASE 1 - END HERE

## PHASE 2 - START HERE

Create the motivation page. basically what is stated under the motivation title in this file. \
Plan the technical part of the research in `research/resources/technical-topics.md`. \
Technical depth is a practical guide: explain the concept, show a small realistic example, move on. \
Assume the reader might only vaguely know Rust, so open with a short Rust primer for C developers. \
Every topic is written as concept first, then C/C++, then Rust. \
Cover mandatory topics C-only bare-metal guides usually skip (toolchain, debugging, testing, FFI, ecosystem and safety standards), and topics unique to Rust (peripheral ownership, typestate, `unsafe`, RTIC/Embassy, panics).

## PHASE 2 - END HERE

## PHASE 3 - START HERE

Write the technical part under `book/src/technical/`, limited to the 3 most important concepts for comparing C and Rust on bare-metal:
1. Hardware access and peripheral ownership.
2. Interrupts and shared state (including DMA).
3. Memory safety, errors and `unsafe`.

No general Rust tutorial (link to the official Rust docs instead) and no "how to read this book" page. \
The reader is technical but may only vaguely know Rust and bare-metal: explain each concept where it is first needed, accessibly, without over-abstracting or simplifying into something untrue. \
The goal is understanding the difference between C and Rust, so every chapter ends with a "C vs Rust at a glance" table. \
All examples use one reference board: ST Nucleo-F401RE.

## PHASE 3 - END HERE

## PHASE 4 - START HERE

Write the "Should We Switch?" chapter (`book/src/decision.md`), as requested in note 3: the reasons for and against switching, based on the 3 technical chapters. \
Short and simple read. \
Anything that needs company data or measurements (our chips, certification needs, team, porting a real module and measuring it, the final recommendation) is marked TBD.

## PHASE 4 - END HERE
