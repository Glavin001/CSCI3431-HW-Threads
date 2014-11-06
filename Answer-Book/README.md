# Homework

This program, x86.py, allows you to see how different thread inter- leavings either cause or avoid race conditions. See the README for details on how the program works and its basic inputs, then answer the questions below.

## Questions

1) To start, let’s examine a simple program, “loop.s”. First, just look at the program, and see if you can understand it: cat loop.s. Then, run it with these arguments:
```bash
./x86.py -p loop.s -t 1 -i 100 -R dx
```
This specifies a single thread, an interrupt every 100 instructions, and tracing of register %dx. Can you figure out what the value of %dx will be during the run? Once you have, run the same above and use the -c flag to check your answers; note the answers, on the left, show the value of the register (or memory value) after the instruction on the right has run.

#### Answer

Simply, the `sub  $1,%dx` subtracts `dx` value by 1, leaving it at -1.
Then when it reaches `test $0,%dx` it is compared to and on `jgte .top` it's comparison result is evaluated, for which it will fail (it is *less* than 0) so it will continue onto `halt`. It will start at `0` (default) and change to `-1` for the next command and stay that way.

| dx |          Thread 0 |
| -- | -- |
| 0 | |
|    -1 |  1000 sub  $1,%dx |
|    -1 |  1001 test $0,%dx |
|    -1  | 1002 jgte .top |
|    -1 |  1003 halt |

2) Now run the same code but with these flags:
```bash
./x86.py -p loop.s -t 2 -i 100 -a dx=3,dx=3 -R dx
```
This specifies two threads, and initializes each %dx register to 3. What values will %dx see? Run with the -c flag to see the answers. Does the presence of multiple threads affect anything about your calculations? Is there a race condition in this code?

#### Answer

Nothing will change for the *inital portion* of the results: they are completely sequential / series, and not run in parallel, which does not create race conditions. The switch will not happen until halt.
However, while there may not be any race conditions, the results for each thread will change. Since the initial value of `dx` has been changed to 3, the `jgte .top` statement will evaluate and cause the "loop" to continue from the top. This will occur 3 times until the value of `dx` is -1 (less than 0).

```bash
$ ./x86.py -p loop.s -t 2 -i 100 --argv=dx=3,dx=3 -R dx -c
ARG seed 0
ARG numthreads 2
ARG program loop.s
ARG interrupt frequency 100
ARG interrupt randomness False
ARG argv dx=3,dx=3
ARG load address 1000
ARG memsize 128
ARG memtrace
ARG regtrace dx
ARG cctrace False
ARG printstats False
ARG verbose False

   dx          Thread 0                Thread 1
    3
    2   1000 sub  $1,%dx
    2   1001 test $0,%dx
    2   1002 jgte .top
    1   1000 sub  $1,%dx
    1   1001 test $0,%dx
    1   1002 jgte .top
    0   1000 sub  $1,%dx
    0   1001 test $0,%dx
    0   1002 jgte .top
   -1   1000 sub  $1,%dx
   -1   1001 test $0,%dx
   -1   1002 jgte .top
   -1   1003 halt
    3   ----- Halt;Switch -----  ----- Halt;Switch -----
    2                            1000 sub  $1,%dx
    2                            1001 test $0,%dx
    2                            1002 jgte .top
    1                            1000 sub  $1,%dx
    1                            1001 test $0,%dx
    1                            1002 jgte .top
    0                            1000 sub  $1,%dx
    0                            1001 test $0,%dx
    0                            1002 jgte .top
   -1                            1000 sub  $1,%dx
   -1                            1001 test $0,%dx
   -1                            1002 jgte .top
   -1                            1003 halt
```

3) Now run the following:
```bash
./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx
```
This makes the interrupt interval quite small and random; use different seeds with -s to see different interleavings. Does the frequency of interruption change anything about this program?

### Answer

This *does* change things. Unlike the previous question, the smaller interval will break the threads into smaller pieces that all manipulate the `dx` register. This will happen whenever the interrupt interval is less than the number of steps required to complete/halt the program.

```bash
$ ./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx -c
ARG seed 0
ARG numthreads 2
ARG program loop.s
ARG interrupt frequency 3
ARG interrupt randomness True
ARG argv dx=3,dx=3
ARG load address 1000
ARG memsize 128
ARG memtrace
ARG regtrace dx
ARG cctrace False
ARG printstats False
ARG verbose False

   dx          Thread 0                Thread 1
    3
    2   1000 sub  $1,%dx
    3   ------ Interrupt ------  ------ Interrupt ------
    2                            1000 sub  $1,%dx
    2                            1001 test $0,%dx
    2                            1002 jgte .top
    2   ------ Interrupt ------  ------ Interrupt ------
    2   1001 test $0,%dx
    2   1002 jgte .top
    2   ------ Interrupt ------  ------ Interrupt ------
    1                            1000 sub  $1,%dx
    2   ------ Interrupt ------  ------ Interrupt ------
    1   1000 sub  $1,%dx
    1   1001 test $0,%dx
    1   ------ Interrupt ------  ------ Interrupt ------
    1                            1001 test $0,%dx
    1   ------ Interrupt ------  ------ Interrupt ------
    1   1002 jgte .top
    0   1000 sub  $1,%dx
    0   1001 test $0,%dx
    1   ------ Interrupt ------  ------ Interrupt ------
    1                            1002 jgte .top
    0                            1000 sub  $1,%dx
    0                            1001 test $0,%dx
    0   ------ Interrupt ------  ------ Interrupt ------
    0   1002 jgte .top
   -1   1000 sub  $1,%dx
   -1   1001 test $0,%dx
    0   ------ Interrupt ------  ------ Interrupt ------
    0                            1002 jgte .top
   -1   ------ Interrupt ------  ------ Interrupt ------
   -1   1002 jgte .top
   -1   1003 halt
    0   ----- Halt;Switch -----  ----- Halt;Switch -----
   -1                            1000 sub  $1,%dx
   -1   ------ Interrupt ------  ------ Interrupt ------
   -1                            1001 test $0,%dx
   -1                            1002 jgte .top
   -1                            1003 halt
```

4) Next we’ll examine a different program(looping-race-nolock.s). This program accesses a shared variable located at memory address 2000; we’ll call this variable x for simplicity. Run it with a single thread and make sure you understand what it does, like this:
```bash
./x86.py -p looping-race-nolock.s -t 1 -M 2000
```
What value is found in x (i.e., at memory address 2000) throughout the run? Use -c to check your answer.

#### Answer

After examining the source code of `looping-race-nolock.s`, I predict the value will be 1, after the execution of the step `add $1, %ax`.

Checking this, I find I am correct:

```bash
$ ./x86.py -p looping-race-nolock.s -t 1 -M 2000 -c
ARG seed 0
ARG numthreads 1
ARG program looping-race-nolock.s
ARG interrupt frequency 50
ARG interrupt randomness False
ARG argv
ARG load address 1000
ARG memsize 128
ARG memtrace 2000
ARG regtrace
ARG cctrace False
ARG printstats False
ARG verbose False

 2000          Thread 0
    0
    0   1000 mov 2000, %ax
    0   1001 add $1, %ax
    1   1002 mov %ax, 2000
    1   1003 sub  $1, %bx
    1   1004 test $0, %bx
    1   1005 jgt .top
    1   1006 halt
```

5)
Now run with multiple iterations and threads:
```bash
./x86.py -p looping-race-nolock.s -t 2 -a bx=3 -M 2000
```
Do you understand why the code in each thread loops three times? What will the final value of x be?

### Answer


The statement `jgt .top` after `test $0, %bx`, is testing if `bx` is greater than 0. Since the intial value of `bx` is 3, and is decremented by 1 with `sub $1, %bx`, this effectively creates a loop with 3 iterations. Both the threads will loop 3 times and each iteration increments the value at address 2000 by 1, therefore the final value is 6 (`2*3*1=6`).

6) Now run with random interrupt intervals:
```bash
./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 0
```
Then change the random seed, setting -s 1, then -s 2, etc. Can you tell, just by looking at the thread interleaving, what the final value of x will be? Does the exact location of the interrupt matter? Where can it safely occur? Where does an interrupt cause trouble? In other words, where is the critical section exactly?

#### Answer

Yes, if you step through the interleaving thread's statements to be executed, you could determine the final value of x. The value should be 2 (when run threads sequentially).
The critical section that must not be interrupted is between `sub $1, %ax` and `test $0, %bx`. This is because the `sub $1, %ax` statement will decrement the `ax` register and if the following `test $0, %bx` is interrupted then the other thread could *also* run `sub $1, %bx`, and that means that whole iteration will be skipped (`bx` went from 2 to 0, for instance, decremented twice, without being compared).

For example:
```bash
$ ./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 0 -c
ARG seed 0
ARG numthreads 2
ARG program looping-race-nolock.s
ARG interrupt frequency 4
ARG interrupt randomness True
ARG argv
ARG load address 1000
ARG memsize 128
ARG memtrace 2000
ARG regtrace
ARG cctrace False
ARG printstats False
ARG verbose False

 2000          Thread 0                Thread 1
    0
    0   1000 mov 2000, %ax
    0   1001 add $1, %ax
    0   ------ Interrupt ------  ------ Interrupt ------
    0                            1000 mov 2000, %ax
    0                            1001 add $1, %ax
    0   ------ Interrupt ------  ------ Interrupt ------
    1   1002 mov %ax, 2000
    1   1003 sub  $1, %bx
    1   ------ Interrupt ------  ------ Interrupt ------
    1                            1002 mov %ax, 2000
    1                            1003 sub  $1, %bx
    1                            1004 test $0, %bx
    1                            1005 jgt .top
    1   ------ Interrupt ------  ------ Interrupt ------
    1   1004 test $0, %bx
    1   1005 jgt .top
    1   1006 halt
    1   ----- Halt;Switch -----  ----- Halt;Switch -----
    1                            1006 halt
```

7) Now use a fixed interrupt interval to explore the program further. Run:
```bash
./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -i 1
```
See if you can guess what the final value of the shared variable x will be. What about when you change -i 2, -i 3, etc.? For which interrupt intervals does the program give the “correct” final answer?

#### Answer

Guess: final value of `2000` is 1.
Correct!

```bash
$ ./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -R ax -i 1 -c
ARG seed 0
ARG numthreads 2
ARG program looping-race-nolock.s
ARG interrupt frequency 1
ARG interrupt randomness False
ARG argv bx=1
ARG load address 1000
ARG memsize 128
ARG memtrace 2000
ARG regtrace ax
ARG cctrace False
ARG printstats False
ARG verbose False

 2000      ax          Thread 0                Thread 1
    0       0
    0       0   1000 mov 2000, %ax
    0       0   ------ Interrupt ------  ------ Interrupt ------
    0       0                            1000 mov 2000, %ax
    0       0   ------ Interrupt ------  ------ Interrupt ------
    0       1   1001 add $1, %ax
    0       0   ------ Interrupt ------  ------ Interrupt ------
    0       1                            1001 add $1, %ax
    0       1   ------ Interrupt ------  ------ Interrupt ------
    1       1   1002 mov %ax, 2000
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1                            1002 mov %ax, 2000
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1   1003 sub  $1, %bx
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1                            1003 sub  $1, %bx
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1   1004 test $0, %bx
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1                            1004 test $0, %bx
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1   1005 jgt .top
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1                            1005 jgt .top
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1   1006 halt
    1       1   ----- Halt;Switch -----  ----- Halt;Switch -----
    1       1   ------ Interrupt ------  ------ Interrupt ------
    1       1                            1006 halt
```

For interrupt intervals greater than or equal to 3 given the correct answer of 2.


8) Now run the same code for more loops (e.g.,set `-a bx=100`). What interrupt intervals, set with the -i flag, lead to a “correct” outcome? Which intervals lead to surprising results?

#### Answer

The "correct" result is 200.
Since there are more loops, now there is more opportunity for the critical section to be interrupted.
What is interesting is that multiples of 3 (3, 6, 9, 12, etc) all give the correct answer of 200. All others give some variation from 100 to less than 200.


9) We’ll examine one last program in this homework (wait-for-me.s). Run the code like this:
```bash
./x86.py -p wait-for-me.s -a ax=1,ax=0 -R ax -M 2000
```
This sets the %ax register to 1 for thread 0, and 0 for thread 1, and watches the value of %ax and memory location 2000 throughout the run. How should the code behave? How is the value at location 2000 being used by the threads? What will its final value be?
Now switch the inputs:
```bash
./x86.py -p wait-for-me.s -a ax=0,ax=1 -R ax -M 2000
```
How do the threads behave? What is thread 0 doing? How would changing the interrupt interval (e.g., -i 1000, or perhaps to use random intervals) change the trace outcome? Is the program efficiently using the CPU?

#### Answer

The value at location 2000 is being used as a flag to determine if the `signaller` or `waiter` should execute.

The first run with `ax=1,ax=0` tells the first thread to be a signaller and the second thread to be a waiter.
The first thread executes and sets the value at location 2000 to 1, which means that the waiter can run.
When the second thread runs, it checks the value at location 2000 to see if it can run: it can, because the signaller has already said it can.

The second run with `ax=0,ax=1` tells the first thread to be the waiter and the second thread to be the waiter. However, the interrupt frequency is defaulting to 50 and therefore the waiter loops through many times before the second thread, representin the signaller, is executed. When the second, signaller, thread finally executes and halt, the first, waiter, thread can now continue and halt.

