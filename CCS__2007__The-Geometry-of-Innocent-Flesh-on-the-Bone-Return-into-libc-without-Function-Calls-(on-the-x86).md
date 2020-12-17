Yinghao Ren (2020/12/7)

- Title: The Geometry of Innocent Flesh on the Bone: Return-into-libc without Function Calls (on the x86)
- Authors: Hovav Shacham
- Venue: CCS 2007
- Keywords: Uses of Short Sequences in Attacks, pop %reg; ret, Discovering Useful Instruction Sequences in Libc, Gadgets

# Paper content
## Summary
The W^X security feature (known as NX/XD in Windows) is a memory protection policy implemented in the operating system which only allows a page to be either writeable or executable, but not both. Due to this feature attackers cannot inject _and_ execute malicious payload into the system (e.g., by using buffer overflow to change the value of return address of a function on the stack). So they will use code that already exists in the process image they are attacking. Since the standard C library, libc is loaded in nearly every Unix program, it is libc that is the usual target which is known as return-into-libc attacks. 
This paper describes a new return-into-libc technique that allows arbitrary computation that does not require calling any functions and in this way removing functions from libc is no help because by ending with the instruction "ret", many instructions can be chained together to perform computations and the libc functions are not needed by attackers. There is an assumption that because of the properties of the x86 instruction set, in any sufficiently large body of x86 executable code there will feature sequences that allow the construction of similar gadgets. So an attacker who controls the stack will be able to use the return-into-libc techniques to cause the exploited program to undertake arbitrary computation by using the gadgets composing of many small instructions. This is called return-oriented-programming.

The procedure of return-oriented programming

1. Find many sequences such as "pop xxx;ret", "push xxx;ret" in the code segment by using some algorithms or some existing tools like RopGadget. 
2. Construct these instructions like "padding + address of gadget 1 + address of gadget 2 + ...... + address of gadget n" to overwrite the return address so it can perform the computation the attackers want.

By using this structure, when the called function returns, it will jump to execute gadget 1. When the execution is completed, the RET instruction of gadget 1 will pop the top data of the stack (that is, the address of gadget 2) to eip, and the program will continue to jump Execute gadget 2, and so on.


In the thoughts part, I will give a specific example.

Using sequences recovered from a particular version of gnu libc, this paper describes gadgets that allow arbitrary computation, introducing many techniques that lay the foundation for return-oriented-programming.

## Strengths
Return-into-libc was considered a more limited attack than code injection because it can only execute straight-line code and attackers can invoke only functions available to them in the program’s text segment and loaded libraries. 
Return-oriented programming successfully comes with a new idea to bypass the limitation of  W^X defense and NX/XD support and come up with more flexible ways to call the libc function.

## Weaknesses
This paper proposes their thesis: 
In any sufficiently large body of x86 executable code, there will exist sufficiently many useful code sequences that an attacker who controls the stack will be able, using the return-into-libc techniques introduced, to cause the exploited program to undertake arbitrary computation.
This is a rough description and it does not give a strict mathematical proof about their thesis. Instead, this paper just  compares the X86 with other ISA like MIPS.

## Thoughts
Originally I just wanted to learn how to call a libc function like "system" 3 times. 2 times is easy. We only need to take advantage of a buffer overflow to make the padding part like “&system1() || &system2() || &/bin/sh1 || &/bin/sh2”(& means the address of the function system() in libc, "/bin/sh" can be found in the code segment by using some tools and if function system() gets the argument"&/bin/sh", it will get the shell. The shell will help attackers do anything they want). The &system1() is on the return address of the function on the stack so when the function finishes executing, it will call system1() in libc whose argument is "&/bin/sh1" and we can get a shell. When this function finishes executing by inputting "exit()", it will call system2() because the &system2() is on the return address of the function system1().  But 3 times is very hard because we can not construct a string like 2 times due to the limitation of argument placement. So I read this paper coincidently not understanding all of them. I got the idea about using "pop xxx, ret". And I made the stack like""&system1() || XXXX || &/bin/sh1 || &system2() || XXXX || &/bin/sh2 || &system3() || XXXX || &/bin/sh3". And find some sequences like"popxxx; ret" to  pop the parameter out. The "XXXX" will be replaced with the address of sequence "popxxx; ret" in the string above. So when finishing the  function system1(), it will execute "popxxx; ret" and pop the &/bin/sh1 out of the stack so the $esp will be added by 4 on 32 bit Linux system then "ret" instruction will pop the &system2() to the $eip and will execute the function to get the shell. Taking advantage of this way, we can even call the function system() in the libc as much as we can.

## Takeaways and questions
I learned the idea of using the sequence existing in the original code. It must be very hard for the defender to limit the attack like that.
