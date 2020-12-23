# Yinghao Ren (2020-12-23)
# Paper information
- Title: The Geometry of Innocent Flesh on the Bone: Return-into-libc without Function Calls (on the x86)
- Authors: Hovav Shacham
- Venue: CCS 2007
- Keywords: Uses of Short Sequences in Attacks, pop %reg; ret, Discovering Useful Instruction Sequences in Libc, Gadgets

# Paper content
## Summary
The W^X security feature (known as NX/XD in Windows) is a memory protection policy implemented in the operating system which only allows a page to be either writeable or executable, but not both. Due to this feature attackers cannot inject _and_ execute malicious payload into the system (e.g., by using a buffer overflow to change the value of return address of a function on the stack). So they will take advantage of the code that already exists in the process image they are attacking by finding the address of these used codes and chaining them. Since the standard C library, libc is loaded in nearly every Unix program, it is libc that is the usual target which is known as return-into-libc attacks. 
This paper describes a new return-into-libc technique that allows arbitrary computation. Let's compare them as below.

The "old" return-into-libc attack:
The attackers aim to modify the return address to point to a function already in memory which often refers to the functions in the dynamic link library libc so it can be limited if some libc functions are removed from the memory by defenders. 

The "new" return-into-libc attack called return oriented programming attack:
By finding short instructions ending with the "ret", they can be chained together to perform computations. So the attackers do not require calling any functions and in this way removing functions from libc is no help for the defenders.

The paper has a core thesis that x86 ISA is extremely dense, meaning that a random byte stream can be interpreted as a series of valid instructions with high probability. Thus for x86 code,  it is quite easy to find not just unintended words but entire unintended sequences of words(It may need another paper review to explain well). So in any sufficiently large body of x86 executable codes, there will exist sequences that allow the construction of gadgets(the target sequence chained together) to perform the target computation. So an attacker who controls the stack will be able to use the return-into-libc techniques to cause the exploited program to undertake arbitrary computation by using the gadgets composing of many small instructions. This is called return oriented programming.

The procedure of return-oriented programming
1. Find many sequences such as "pop xxx;ret", "push xxx;ret"（the two kinds of instructions are the most commonly used because "push" instructions can help store the value of a variable on the stack and "pop" can transfer the value to the register so that CPU can use registers to perform the computation） in the code segment by using some algorithms or some existing tools like RopGadget. 
2. Construct the payload like "padding (padding is to fill the memory between the start address like the array and the return address ) + address of gadget 1 + address of gadget 2 + ...... + address of gadget n" to overwrite the return address so it can perform the computation the attackers want. It needs some practical experience
By using this structure, when the called function returns, it will jump to execute gadget 1. When the execution is completed, the ret instruction of gadget 1 will pop the top data of the stack (that is, the address of gadget 2) to $eip(the register which stores the address of the next instruction), and the program will continue to jump execute gadget 2, and so on.

In the thoughts part, I will give a specific example to explain the whole procedure.

Using sequences from a particular version of gnu libc, this paper describes how to construct gadgets that allow arbitrary computation like how to perform some basic computations(add, sub, store, load) and they lay the foundation for return-oriented-programming.

## Strengths
Return-into-libc was considered a more limited attack than code injection because it can only execute straight-line code(the attackers can only call function one after another instead of more flexible behavior like branch) and attackers can invoke only functions available to them in the program’s text segment and loaded libraries. 

Return-oriented programming successfully comes with a new idea to bypass the limitation of  W^X defense and NX/XD support and come up with more flexible ways to call the libc function.
​
## Weaknesses
This paper proposes their thesis: 
In any sufficiently large body of x86 executable code, there will exist sufficiently many useful code sequences that an attacker who controls the stack will be able, using the return-into-libc techniques introduced, to cause the exploited program to undertake arbitrary computation.
This is a rough description and it lacks specific descriptions of the feasibility of ROP on other ISA(Whether it is Turing-Complete and I do not understand this part very clearly). Instead, this paper just roughly compares the X86 with other ISA like MIPS about the existence of unintended instructions.
​
## Thoughts
If we only allocate an Array on the stack with m memory units, the value stored above the Array memory unit on the stack is the value of caller function's $ebp and above that is the return address of callee's function. We can assign a string whose size is bigger than m memory units. If we control the overflow part well by designing a payload, we can change the return address of the callee's function. Originally I just wanted to learn how to call a libc function like "system" 3 times. 2 times is easy. We only need to take advantage of a buffer overflow to make the payload like “&system1() || &system2() || &/bin/sh1 || &/bin/sh2”(& means the address of the function system() in libc, "/bin/sh" can be found in the code segment by using some tools and if function system() gets the argument"&/bin/sh", it will get the shell.  The shell will help attackers do anything they want, || means the Interval symbol). The &system1() is on the return address of the function on the stack so when the function finishes executing, it will call system1() in libc whose argument is "&/bin/sh1" and the argument can help us get the shell. When this function finishes executing by inputting "exit()", it will call system2() because the &system2() is on the return address of the function system1().  But 3 times is very hard because we can not construct a string like 2 times due to the limitation of argument placement. (It needs practical experience and I cannot describe that well)
I got the idea about using "pop xxx, ret". And make payload like""&system1() || XXXX || &/bin/sh1 || &system2() || XXXX || &/bin/sh2 || &system3() || XXXX || &/bin/sh3". Besides find some instruction sequences like"pop xxx; ret"(xxx is the name of a register) to pop the parameter out. The "XXXX" will be replaced with the address of sequence "pop xxx; ret" in the string above. So when finishing the function system1(), it will execute "pop xxx; ret" and pop the &/bin/sh1 out of the stack so the $esp(a register that stores the top address of the function callee stack) will be added and stores the address of  &system2() then "ret" instruction will pop the &system2() to the $eip and will execute the function to get the shell. Taking advantage of this way, we can even call the function system() in the libc as many times as we can.

## Takeaways and questions
I learned the idea of using the sequence existing in the original code. It must be very hard for the defender to limit the attack like that.
