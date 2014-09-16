---
layout: post
title: Playing with Rust part 3
---

Part 3! Trying a few other modern (or not so modern) languages.

Yesterday, I embarrassed myself by showing my busy beaver to Felix Klock at the Paris Rust
User Group, just to be told that rustc had a -O option... now the rust beaver is even one
tad faster than the C version. At least this shows I managed to get the actual code right.

I also implemented the 5-state beaver in scala, go, and swift.

* rust: 0.19 second (with -O)
* C: 0.2 second (clang, with -O2)
* go: 0.4 second
* scala: 0.6 second
* swift: TLDR

The first surprise was to see scala so close to C, go and rust. I would have expected it to be
way slower. I'm a seasoned scala developer, but I did not have to invoke any kind of
witchery to get this result. Picking "Buffer" for the tapes implementation seems to be 
the right pick, but it is also the natural one. There seem to be little variation of
execution time as time goes, the JIT compiler is already on top of the game at the first iteration.

I was also surprised to have the go version so fast without having to work at all on
the tape re-allocation. The append method seems to do the right thing out of the box.

As for swift, I was amazed to see how badly it behaves. I called an apple savvy friend for help,
but so far, his efforts have not paid. I can only assume the recent changes in array semantics and
their consequences have not been digested yet.

All in all, I don't think I need to say I'm really, really impressed by Rust. And it sounds like
the team start to feel confident about a freeze and a 1.0. This is great news. We may dodge a go-dominated
world after all...
