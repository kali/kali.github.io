---
layout: post
title: Playing with Rust part 2
---

Part 2 of my adventures with Rust and the busy beaver.

Performance
-----------

Rust aims at delivering the same performance than C.
[I tried it.](https://github.com/kali/rust-sandbox/blob/f9e32abf1b3fc94938dfee286d183f5e6b6e2286/busy-beaver-c/beaver.c)

On my test machine (my new Macbook pro), the C implementation runs in 500ms and the rust one in 2.190s. Ouch.

Well, my implementation is certainly naive, so let's try to improve that. First, I tried to discard some of the
possible source of bias by running the computation 10 or 100 times in a loop, with no significant difference.
And being a java/scala long time user, believe me, I consider that good news.

The other good news is, as the binary is just that, a binary, all standard profiling tools will work, starting
with instruments. I recompiled with -g and got a nice .dSYM directory, and ran the beaver through instruments
(with a 10 times loop).

After a few clicks in Instruments, I managed to get that:

<a href="/assets/2014-09-03-Instruments-1.png">
    <img src="/assets/2014-09-03-Instruments-1.png" alt="First profile" height="400px"></img>
</a>


