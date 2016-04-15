Threads are a [general purpose] abstraction.
It's a state machine implemented using:

- stack frame
- thread locals
- registers
- sequence of instructions
- a program counter

2:30:
https://www.youtube.com/watch?v=wxXIbaJBZlE

=> if you know the problem you are working on, you can do better than a general purpose solution

---

"mutex" is short for "mutual exclusion semaphore".  A mutex is
a kind of semaphore.  Usually when people refer to semaphores, though, they
mean counting semaphores.  Often people use the term as if that's the only
kind of semaphore that exists.  Confusion abounds.

