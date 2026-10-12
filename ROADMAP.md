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

1. Create the file tree. differ resources and notes from the actual product.
2. Create the main page, single opening sentence.

## PHASE 1 - END HERE
