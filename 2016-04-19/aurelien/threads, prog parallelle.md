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
Scott Meyers Effective C++11

1:00:00
Un std::thread peut être joignable ou pas. S'il est joignable (c.a.d qu'il correspond réellement à un thread d'exécution de l'OS sous-jacent qui peut être joint) et que le destructeur de std::thread est appelé, alors std::terminate() est appelé. C'est une différence avec Boost (avant 1.50) où detach était alors appelé.

Il faut donc s'assurer que ses thread deviennent injoignables dans *tous* les chemins d'exécution. Malheureusement pas de classe RAII pour std::thread dans la lib standard. Boost en propose un (1:11:20).



---

"mutex" is short for "mutual exclusion semaphore".  A mutex is
a kind of semaphore.  Usually when people refer to semaphores, though, they
mean counting semaphores.  Often people use the term as if that's the only
kind of semaphore that exists.  Confusion abounds.

