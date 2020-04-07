---
title: "Low-Level Software Security by Example"
layout: post
date: 2020-04-07 00:42
tag:
- security
- c
headerImage: false
category: blog
author: junhao
description: N.A.
---

> This is a paper review for "Low-Level Software Security by Example", by Úlfar Erlingsson, Yves Younan, and Frank Piessens. The authors present a selection of low-level attacks on C software and the corresponding defense mechanisms as well as the evaluation.


## Example 1

#### Attack: Corruption of a Function Return Address on the Stack

By overflowing the input buffer, the attacker can overwrite function's return address in order to direct execution to code of their choice. It is a common case that the attack payload is embedded in the input buffer itself, and typically takes advantage of machine code op bytes `0xcd` & `0x2e` (for syscall) and `0xeb` & `0xfe` (for jmp). A real life example exploiting such vulnerability was the Blaster worm in 2003.

#### Defense: Checking Stack Canaries on Return Address

One way to preserve the integrity of return address is to place a public dynamic canary value right above the buffer (and thus) below the return address and the saved base pointer. This value is checked upon returning, so if it gets corrupted it signals an address clobbering. It is simple and effective, but it does not prevent the function from executing.

## Example 2

#### Attack: Corruption of Function Pointers Stored in the Heap

Similar to buffer overflow on the stack, pointers stored in the data segment can be corrupted as a result of the buffer overflow. As a concrete example, consider the following snippet, where the attacker can overflow the `cmp` pointer using strings one and two.

{% highlight C %}
typedef struct _vulnerable_struct
{
    char buff[MAX_LEN];
    int (*cmp)(char*,char*);
} vulnerable;

int is_file_foobar_using_heap( vulnerable* s, char* one, char* two )
{
    // must have strlen(one) + strlen(two) < MAX_LEN
    strcpy( s->buff, one );
    strcat( s->buff, two );
    return s->cmp( s->buff, "file://foobar" );
}
{% endhighlight %}

#### Defense: Making Data not Executable as Machine Code

This is easy to understand. In the context of code above, once the program counter reaches the first byte of the attack payload, the attack will fail to execute.

## Example 3

#### Attack: Execution of Existing Code via Corrupt Pointers

This class of attacks is also known as *jump-to-libc* and is attractive on architecture where input data cannot be directly executed as machine code, because it works by redirecting execution to a series of existing library functions to serve the malicious purpose. As an example, suppose the program takes in a pointer function and executes it. The attacker can supply a pointer that points to an executable memory address in a system library. In our example, the assembly version of the machine code will be `mov esp, ebx`, which can be used to set the stack pointer to the start address of the input buffer and changes the program counter to the address of the second integer in the input buffer. That second integer also contains an executable memory address, and upon return the thrid integer is yet another executable memory address. Hence, a chain of system library functions can be executed, which may allocate a writable and executable memory for the attack payload without directly execution.

#### Defense: Enforcing Control-Flow Integrity on Code Execution

For C/C++ software, any indirect control transfers, such as through function pointers or at return statements, will have only a small number of valid targets. So with CFI security policies, one can dictate the the program execution must follow  a path of a control-flow graph that is determined ahead of time.

## Example 4

#### Attack: Corruption of Data Values that Determine Behavior

Without diverting the execution path, attacker can launch *data-only attacks* in some scenarios. For instance, if a program takes an input string as a command argument supplied to be executed, although the "safe" command can be hardcoded in the program, it is still possible that the attacker can change the "safe" command to a malicious one once the address is correctly guessed.

#### Defense: Randomizing the Layout of Code and Data in Memory

To bar attackers from the knowledge of data addresses, Address-Space Layout Randomization (ASLR) can be employed. This is a general idea that can be implemented in various ways. For instance, when the OS reboots, the system libraries are located sequentially in memory in the order they are needed, but at a starting point chosen randomly from 256 possibilities. In addition, the memory layout can be shuffled at a finer granularity by padding an unused memory between some or all stack frames.

## Summary of Defense Effectiveness

<table border="1px solid" cellspacing="0" cellpadding="6" bordercolor="#c2c2c2" align="center">
    <tr>
        <td></td>
        <td>Return address corruption</td>
        <td>Heap function pointer corruption</td>
        <td>Jump-to-libc</td>
        <td>Non-control data</td>
    </tr>
    <tr>
        <td>Stack canary</td>
        <td>Partial defense</td>
        <td></td>
        <td>Partial defense</td>
        <td>Partial defense</td>
    </tr>
    <tr>
        <td>Non-executable data</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td></td>
    </tr>
    <tr>
        <td>Control-flow integrity</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td></td>
    </tr>
    <tr>
        <td>Address space layout randomization</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
        <td>Partial defense</td>
    </tr>
</table>
<br>