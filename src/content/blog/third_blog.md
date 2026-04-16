---
title: 'Handling Instruction Branching in Modern Processors'
description: 'Here we together go through what instruction branching is, how it is handled and how you never knew your processor was this locked in so that you can play mario cart'
pubDate: 'Apr 16 2026'
heroImage: https://images.unsplash.com/photo-1625315714730-d0830cd368bd?q=80&w=1332&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---
Before anything else let's establish one simple thing, which is `everything you do in your system is an instruction to the processor`.
Whether it is playing games, writing code, running that code, using Microsoft Paint....everything is an instruction to the processor.

Now since everything is an instruction then naturally we need a good and efficient method to deal with the processing of these instructions.

And that is what we are going to explore today...

## Let's Start With Some History Before We Dive Into The Goodies
Earlier in the biblical days the processor use to execute instructions sequentially, there was a simple loop without any parallelism or any overlap.
It looked like this:

`Sequential Fetch -> Decode -> Execute -> Repeat`

Let's look at the following code example to gain more clarity.
```c
while(cpu_running){
// Fetch
uint8_t opcode = memory[PC];
PC++;
// Decode
Instruction instr = decode(opcode);
// Execute
execute(instr);
}
```
The CPU and Memory use to share the same bus for both data and instructions(Von Neumann Architecture)
```
Clock:  |--FETCH--|--DECODE--|--EXECUTE--|--FETCH--|--DECODE--|--EXECUTE--|
         instruction 1                    instruction 2
```
### The Von Neumann Bottleneck
```c
int sum = 0;
for(int i = 0; i < 1000; i++){
    sum += i;
}
```
On early hardware, the above code compiled to;
```asm
; 8080-style assembly
LOOP:
    MOV A, M       ; FETCH from memory → wait for memory
                   ; DECODE
                   ; EXECUTE
    ADD B          ; FETCH opcode → wait again
                   ; DECODE
                   ; EXECUTE
    INR C          ; again... full cycle
    JNZ LOOP       ; again... full cycle
```
`Each instruction = a full memory round trip`. Memory was slow (~hundreds of ns). CPU sat idle waiting.

---
---

## Intel 8086 : The First Step Towards a Better Life(1978)

8086 introduced `Bus Interface Unit(BIU)` and `Execution Unit(EU)`, the first ancestors to modern ICU and EU (I'll discuss them later in the blog in detail so relax).

```c
┌─────────────────────────────────────────┐
│              Intel 8086                 │
│                                         │
│  ┌────────┐      6-byte      ┌───────┐  │
│  │  BIU   │ ──► prefetch ──► │  EU   │  │
│  │(fetch) │     queue        │(exec) │  │
│  └────────┘                  └───────┘  │
│   fetches ahead              executes   │
│   while EU works             current    │
└─────────────────────────────────────────┘
```
Now how these two new units changed the game will be easily explainable through the following example;
```c
// What the 8086 BIU/EU split looked like conceptually

// BIU thread (runs in parallel with EU)
void biu() {
    while (queue_not_full()) {
        queue.push(memory[PC++]);  // prefetch instruction bytes ahead of time
    }
}

// EU thread
void eu() {
    while (true) {
        uint8_t opcode = queue.pop();  // reads from queue, not memory directly
        Instruction instr = decode(opcode);
        execute(instr);
    }
}
```

So to put it in very simple terms -> `you just keep fetching the instructions, keep them in a queue until it isn't full and let the execution unit deal with the execution part`.

But instead of this smart advancement there were still problems remaining

### Challenges That Remained
#### 1. Branch Problem - prefetch gets thrown away
```c
if (x > 0) {
    foo();   // BIU prefetched instructions after this branch
} else {
    bar();   // but we jump here — all prefetched bytes = wasted
}
```
- BIU prefetched:  ***`[foo_instr1] [foo_instr2] [foo_instr3]...`***
- Branch taken to: **bar()** ← queue flushed, *start over*
- Wasted cycles:   ~4–8 cycles minimum (huge in that era)

#### 2. Memory Latency Was Still a Bottleneck
```asm
MOV AX, [BX]   ; load from RAM
ADD AX, CX     ; EU had to wait — no cache existed on early chips
MOV [DI], AX   ; store — another wait
```
No L1/L2 cache on early processors. Every memory access = real RAM access.

#### 3. No Out-of-Order Execution
```c
int a = x + y;   // instruction 1
int b = p + q;   // instruction 2 — independent of instruction 1
                 // early CPUs still waited for instr 1 to finish
                 // because they couldn't detect independence
```
---
---

## The Goodies
The modern processors are ***`superscalar`*** meaning that they can handle multiple instructions and they handle them out-of-order, meaning that the order of execution is not similar to the order of instructions in the machine-level code.

The overall design has 2 main parts:
- Instruction Control Unit (ICU)
- Execution Unit (EU)

Let's go through them one by one and you'll know how a processor handles instructions.

### Instruction Control Unit (ICU)

The following block diagram will help you understand things better

![The Microprocessor Layout](/public/mp-layout.png)

Let's break this down real quick.

The `ICU` is the top block. Inside it, the `Fetch Control` unit is the one 
firing requests to the `Instruction Cache` asking for instructions. Once the 
cache hands them over, they go straight into `Instruction Decode` which breaks 
them down into simple operations the processor can actually work with.

Now there's also a `Register File` in there which holds the current values of 
all your registers, and a `Retirement Unit` which is basically the guy who 
checks at the end -> *"hey did our branch prediction turn out to be correct??"*. 
If yes, great. If not, everything gets rolled back.

Once the ICU decodes the instructions into operations, they get passed down to 
the `EU` — the bottom block. The EU has multiple `Functional Units` running 
in parallel: one for handling branches, two for arithmetic, one for loading 
data from memory and one for storing it back. This is what makes modern 
processors ***superscalar*** — multiple things happening at the same time.

All the load/store units talk directly to the `Data Cache`, which is just 
fast memory sitting close to the processor so it doesn't have to go all the 
way to RAM every time.

The task of this unit is to `fetch the instructions` from an ***instruction cache*** which is a fast memory that stores the recently accessed instructions.  
In general the ICU prefetches the instructions while the previous ones are being executed: this is done to ensure efficiency i.e. the processor don't have to sit ideal while instructions execute.

Now comes the golden child of our discussion: `The Branch Problem`  
When a program hits a *branch*, there are two possible directions a program might go:
- The branch can be *`taken`*.
- The branch can *`not be taken`*.

#### But wait I forgot to tell you about branching!!

Well branching for a program is when, during the execution of the program, the control shifts to some other memory address.

For Example: A return statement in a function, a conditional jump that resides inside an if block e.t.c

#### Easy Right ??.... Alright then back to the main discussion :)
Branching can be of two types:
- 1. Conditional Branching.
- 2. Non-Conditional Branching.

Now let's look at each of these one by one and understand what's happening

### Conditional Branching

This is when the processor knows the *`destination`* but doesn't know the *`direction`*.  
Meaning that the information is hard coded in the instruction itself which tells where to go if this branch is taken but whether this branch will be taken or not...this is not known to the processor.  

So what does the processor do then??  
Simple, it makes a `prediction` based on previous behaviours, it is called `branch prediction` in which it guesses whether or not a branch will be
taken and also predict the target address for the branch. Let's assume that the prediction says that `yes..there will be a branch hit`, then the processor starts loading the further instructions which need be performed when the branch hits.  

But wait...*predictions aren't always right*. So what if there is no branch hit, the procesor already loaded the instructions at the destination..what will happen to that??  
Well all that work will be simply erased and the processor will prepare instructions at the new destination for `Execution Unit` to execute.

### Non-Conditional Branching

This is when the processor knows the *`direction`* but it isn't aware of the *`destination`*.  
Meaning just the opposite of what you just read dude...This is specifically called `indirect branching` — the processor knows a branch 
*will* happen but the destination address is only known at runtime, sitting 
inside a register or memory location.  

**For example**: a direct jump like jmp 0x400 always has a known destination. The "unknown destination" case is specifically indirect branches (jmp rax, ret, call [rax]).

So how does the processor handle this??

It uses a small hardware stack called the `Return Address Stack (RAS)`. Every 
time a `call` instruction is made, the return address gets pushed onto the RAS. 
When the processor hits a `return`, it just pops from the RAS and that's your 
destination. Simple and fast.

But RAS only works for `return` statements. What about indirect jumps like 
`jmp rax` where the destination is sitting in a register?? For those, the 
processor uses a `Branch Target Buffer (BTB)` — basically a cache of 
*"last time I was at this instruction, I jumped to this address"*. So it 
predicts the destination based on history.

And just like branch prediction — if it gets it wrong, everything speculatively 
loaded gets flushed and the processor starts over from the correct destination. 
Painful but rare when the predictor is well trained.

---

## Alright Let's Wrap This Up

So here's what we covered today:

Back in the day processors were doing everything sequentially, one instruction 
at a time, and the CPU just sat there waiting for memory like it had nothing 
better to do. Then the 8086 came along and split fetch and execute into two 
separate units — a genuinely big deal at the time.

Fast forward to modern processors and we have full-blown superscalar machines 
with an ICU handling fetch, decode and branch prediction, and an EU running 
multiple functional units in parallel. The whole thing is basically an 
extremely optimized guessing machine that is right most of the time.

Branching is the tricky part — conditional branches have a known destination 
but unknown direction, non-conditional indirect branches have a known direction 
but unknown destination. The processor handles both through prediction and 
hardware stacks, and when it gets it wrong it just flushes and moves on like 
nothing happened.

That's your processor. Quietly predicting the future at billions of cycles per 
second so you can play Mario Kart without lag.