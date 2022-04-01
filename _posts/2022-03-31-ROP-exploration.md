---
published: true
layout: post
---
Currently studying/reading up on ARM assembly, and moving on to exploits -- this week I've read up on stack overflows and some basic Return Oriented Programming (ROP), following [this](https://azeria-labs.com/return-oriented-programming-arm32/) great tutorial on azerialabs. I suppose partially due to my complete lack of knowledge of this entire field, actually getting the exploit to work on my end was a lot more difficult than I felt it should've been, especially after reaching the endpoint (kind of like a hard math problem which seems a lot easier in hindsight, but when you were completely lost when trying to solve originally)

To run/write ARM code, I've been using the [Azeria Labs VM](https://azeria-labs.com/lab-vm-2-0/) on VirtualBox (I believe the intented software was VMware, so I had a LOT of trouble trying to get this to run on VirtualBox, but finally succeeded in the end)

This post will specifically touch on Challenge2 in the very first link, which comes included in the VM. My thought processes will probably assume one has, at the very least, some understanding of ARM syntax and the stack, since that is roughly what I had while trying to crack this. (For a decent runthrough of these concepts, do peruse the azerialabs website)

Another note is that the solution is highly Google-able as it is a very basic form of a ReturntoLibc attack. I suppose what I will document here are my mistakes and ideas that I entertained -- how I'd explain the whole process to myself.

--------------------------------------------------

## **Buffer Overflows?**

So sometimes a program creates a buffer to contain a certain input from the user. 

	void Func1(char* input) {
    	char buffer [80];
        strcpy(buffer, input);
    }


However, in a language like C, there isn't really a default system in place to prevent users from entering inputs **larger** than the buffer. So in the case of that, the excess input will **overflow** into other parts of memory, and it turns out that it is possible to control the content of what exaxctly overflows to those parts of memory.

Borrowing this image from AzeriaLab's writeup on this,
![stackoverflow.png]({{site.baseurl}}/assets/images/stackoverflow.png)
_Credit: Azeria Labs (https://azeria-labs.com/stack-overflow-arm32/)_


The buffer content can actually overflow into the stack, which, among other things, is the place which saves the return address for a program when it branches into a function. Hence, by rewriting this return address into something else, we can get the program to just do something completely different after completing the function. Do read the [aforementioned post](https://azeria-labs.com/stack-overflow-arm32/) for greater detail.



--------------------------------------------------

## **What next?**

The goal in both challenges is to exploit the C file, and gain access to the [shell](https://en.wikipedia.org/wiki/Shell_(computing)).
![Screenshot 2022-03-30 at 8.45.20 PM.png]({{site.baseurl}}/assets/images/Screenshot 2022-03-30 at 8.45.20 PM.png)


The solution to the first one was to overflow the buffer, and rewrite the PC to point to the stack, and from there execute shellcode (in this case, code that we have manually inserted to run the command "system(/bin/sh)" and gain access to the shell)

Now, this was all possible because the stack in Challenge1 had RWX permissions, ie Read, Write and Execute permisions.

In Challenge2, the XN (Execute Never) bit was turned on, and it is not longer as straightforward to just straight up run whatever code the attacker wants to inject. What we now want to do is to kind of piece together puzzle pieces, using code snippets that are already present in the system, in order to carry out what we want. Specifically here, we will be using code snippets from the libc (basically the preinstalled C library) in order to carry out our evil deeds, since libc has execute permissions.

![The stack only has read and write permissions.]({{site.baseurl}}/assets/images/permissions.png)


_The stack only has read/write permissions, whereas part of libc does have execute permissions._


(While it seems dubious to hope that the code snippets you find in a system/machine can actually be pieced together to do what an attacker wants, it has been proven that ROP is Turing-Complete (that as a whole, the code snippets you get basically function like any other programming language out there))

Now, what we have to do first is to determine the **offset** of the program, ie how much it takes to overflow the buffer.

Fortunately, we can use our debugging tool (GDB) to figure this out instead of trial and error of running a massive number of As into the program.

![conveniently getting a massive output that we can paste into the program]({{site.baseurl}}/assets/images/offset1.png)


_Generating a long string to crash the program with_

After this, we run the program through and observe the segfault that occurs, and do the following to find the offset:

![Running pattern search to determine at what point stuff overflows into the PC.]({{site.baseurl}}/assets/images/offset2.png)


_Running pattern search to determine at exactly what point stuff overflows into the PC (Or rather what part of the code is popped into PC from the stack)_

(I am yet to fully grasp why we search for $pc + 1, but roughly I believe it is because in ARM the PC always adjusts itself for alignment purposes)



Anyways, we have discovered that the offset is 132, ie our 'payload' will begin with 132 characters of garbage before getting to the actual meat.

So, we have the beginnings of our exploit code:

	#!/usr/bin/python
    from struct import pack
    payload = 'A' * 132
    print payload

-------------------------------------------------------------
## **The actual ROPping**

Now, we need to figure out what code snippets to use in order to make the machine run system("/bin/sh").

Roughly, our final goal is:


**1) Have the PC pointing to the 'system' function, wherever it is in memory.**


**2) Have r0 contain the string "/bin/sh" (which, yes, is present in libc)**
_(why r0 specifically? Because the system function runs whatever is in r0 as the argument)_
  

That actually doesn't seem that hard! Let's begin with



### **Where are system and "/bin/sh"?**


![system.png]({{site.baseurl}}/assets/images/system.png)


_It's as simple as that._


By running the code in the debugger, setting a breakpoint, and using the above command, we find that the address of system is at **0xb6f0b48c**.


![binsh.png]({{site.baseurl}}/assets/images/binsh.png)


Next, we run this command (googleable) and we find that the string "/bin/sh" is stored in **0xb6fb407c**.


### **How do we control the registers?**


Now for the hard part. How do we get r0 and PC to point to what we require?


For r0, the **LDR** function comes to mind. We can try some sort of "**LDR r0, [SP]**" gadget if that exists (gadget = code snippet)


Time for our good friend Ropper. Cd-ing into the directory where libc resides, we run Ropper and use the command "**search /1/ LDR r0**" to search for any gadgets of length 1 (excluding the return instruction). _(Length 1 ensures that we minimise the number of instructions and thus the amount of complications we have to bother with)_


![ropper1.png]({{site.baseurl}}/assets/images/ropper1.png)
![ropper2.png]({{site.baseurl}}/assets/images/ropper2.png)


_The list is longer than this, but we are only interested in loading something from what SP is pointing to onto r0, not data from other registers._


Damn, looks like we aren't in luck. There is no **LDR r0, [SP]**. But isn't the **LDR r0, [SP, #4]** gadget the same thing? I just have to make "/bin/sh" appear 4 bytes further down. _Voila,_ I thought, _this is it._ We need the address of this gadget, which isn't trivially the number we see in this screenshot. We need to add it onto the base address of libc, which I have indicated as follows:
![libcwhere.png]({{site.baseurl}}/assets/images/libcwhere.png)


_We will use the python function struct pack later on to add the addresses together because we're lazy._



At that time, my idea was:


![stack1.png]({{site.baseurl}}/assets/images/stack1.png)
![stack2.png]({{site.baseurl}}/assets/images/stack2.png)


_The stack, and the corresponding hex representation_


Where the flow of events I expected was: pop gadget onto pc --> load r0 with "/bin/sh" --> next on the stack is system() so PC will execute system with r0 as the argument.

-----------------------------------------------------------------------

## **Back to the payload!**

Now, translating all that into the payload:

	#!/usr/bin/python
    from struct import pack
    libc = 0xb6ede000 #Base address of libc
    
    payload = 'A' * 132
    payload += pack('<I', libc + 0x000259d3) #The gadget LDR r0, [SP, #4]
    payload += "/x8c/xb4/xf0/xb6/x7c/x40/xfb/xb6"
    print payload

------------------------------------------------------------------

### **Tangent: what's with the hex (/x8c/xb4/...)?**


It may not seem completely obvious how exactly we went from the addresses we found to...  this weird string of alphanumeric characters. Basically:


-The "/x" indicates that the following two characters are in hex(adecimal). So /xb6 really just means 0xb6. The reason they are separated into 2 characters at the time, I suppose, is becaue we need to feed the machine the info/input byte by byte (really have no clue on exactly why, though.)


-**Why is it in reverse?** If you noticed, we wanted the machine to read 0xb6f0b48c and then 0xb6fb407c, but each of these hex numbers has been split into 4 individual bytes, had the order reversed, and then joined together. This is due to a feature of the system known as little-endianness. Basically when the system reads a 4 byte chunk, it treats the byte at the lowest memory value as the **least significant byte (LSB)**, ie, it is at the lowest 'place' (Think ones place, and tens place in Base 10). So, the byte on the top of the stack is actually the '16's and '1s' place of the eventual 4-byte chunk we want to feed the machine. And when our payload overflows into the stack, it does so from the top down, meaning that whatever goes in first will be at the top of the stack, and thus the LSB/rightmost part of the 4-byte hex value we desire. 

![Little-Endian.svg]({{site.baseurl}}/assets/images/Little-Endian.svg)


_A nice graphic that I found on Wikipedia. (Credit: R. S. Shaw on https://en.wikipedia.org/wiki/Endianness)_

------------------------------------------------------------------------

Perfect! Now let's run it:

	user@azeria-labs-arm:~/challenges$ chmod +x exploit2demo.py #making the script executable
    user@azeria-labs-arm:~/challenges$ export BOOM=$(./exploit2demo.py) #setting the environment variable BOOM to run the code (which basically does the same thing as setting a very long string as an argument for the program)
    user@azeria-labs-arm:~/challenges$ ./challenge2 "$BOOM" #running the program with the 		payload...
    Segmentation Fault
    user@azeria-labs-arm:~/challenges$
    
------------------------------------------------------------------------

## **Wait, what?**


No shell access! The program only crashed, and did nothing else. What went wrong?


After a fair bit of debugging, I realised the issue: The payload I sent in fact did almost everything I wanted it to do: r0 really was holding the value "/bin/sh", and the top of the stack really was pointing to system.

![righttrack.png]({{site.baseurl}}/assets/images/righttrack.png)
_Signs that I was on the right track._

The issue that I overlooked was that I assumed that the program flow would just go back to SP after loading r0 with my desired value. However, the command that followed **LDR r0, [SP, #4]** was actually **BLX r4**, and r4 was pointing at **0xbefffa88**, effectively skipping my buffer overflow! Back to square one, I suppose. 


### **[Author's note]**


While typing this out, I considered that since r4 was pointing, like, 20 bytes away from my desired function call, I could've just inserted an additional 20 characters of garbage instead. As follows:

	#!/usr/bin/python
    from struct import pack
    libc = 0xb6ede000 #Base address of libc
    
    payload = 'A' * 132
    payload += pack('<I', libc + 0x000259d3) #The gadget LDR r0, [SP, #4]
    payload += 'A' * 4
    payload += "/x7c/x40/xfb/xb6"
    payload += 'A' * 24
    payload += "/x8c/xb4/xf0/xb6"
    print payload

However, this still didn't work. Some examination revealed that since the BL swapped the program to Thumb mode, the PC got sent to the wrong place (1 byte away from the system() call) and the system crashed from there as the address didn't align to Thumb. This requires some further investigation as I'm not so confident on my understanding of this.

------------------------------------------------------------

## **Back to Square One?**


So, I couldn't find a satisfying **LDR r0, [SP]** instruction. Were there any alternatives? After thinking for a bit, I realised that I could try, instead, **popping** system() and "/bin/sh" off the stack into r0 and pc. So, lets open up Ropper again and do "**"search /1/ pop{r0"**.

![ropper3.png]({{site.baseurl}}/assets/images/ropper3.png)
_The perfect gadget?_


And straight off the bat we see that there is a **pop {r0, pc}** gadget! Now we can alter our code to include this. Let's first check out the new flow that we require:


![stack3.png]({{site.baseurl}}/assets/images/stack3.png)
![stack4.png]({{site.baseurl}}/assets/images/stack4.png)


_Notice that bin/sh and system() are inverted compared to the previous try_


Now, our payload looks like:

	#!/usr/bin/python
    from struct import pack
    libc = 0xb6ede000 #Base address of libc
    
    payload = 'A' * 132
    payload += pack('<I', libc + 0x0000726d) #The gadget pop {r0, PC}
    payload += "/x7c/x40/xfb/xb6/x8c/xb4/xf0/xb6"
    print payload

Lets try it out (Please work)

![]({{site.baseurl}}/assets/images/failure.png)


_Segmentation Fault... again._


Another failure? Yet the objective seems in reach --- the next instruction should've been to carry out the system() command already. What happened?


As it turned out, I did not realise what I needed to do to fix this until I actually cracked it with another gadget. For the sake of not prolonging this much further, I'll cover the small mistake I made.

------------------------------------------------------------

## **Finally?**


All I had to do was to look at $PC.
![theerror.png]({{site.baseurl}}/assets/images/theerror.png)



The address that PC was pointing at and the actual next address for system were **not the same**. This is because the program entered thumb mode and everything was displaced by 1. As such, PC was not pointing to a complete instruction and crashed. All I needed to do was to change that /x8c to /x8d.

	#!/usr/bin/python
    from struct import pack
    libc = 0xb6ede000 #Base address of libc
    
    payload = 'A' * 132
    payload += pack('<I', libc + 0x0000726d) #The gadget pop {r0, PC}
    payload += "/x7c/x40/xfb/xb6/x8d/xb4/xf0/xb6" #just the slightest of differences...
    print payload


![success.png]({{site.baseurl}}/assets/images/success.png)


_There we go._



That was quite the marathon (for me), although I felt like this really would be so straightforward for many others. I guess we all have to start somewhere.


--------------------------------------------------------------

## **BONUS: So what did I end up doing?**


Inspired by a source over the internet, I went to look for longer instructions that contained LDR r0, using "**search /2/ LDR r0**". 

![questionable.png]({{site.baseurl}}/assets/images/questionable.png)


_Desparately grasping at straws._


I ended up using the instruction at **0x000beb2e**, which was **LDR r0, [SP, #4]; add SP, #8; pop {r4, PC}**. This required a bit more acrobatics down on the stack, but everything still worked fine. Eventually, I realised that my system() address was off by one byte, as mentioned above, and the rest is history.

![stack5.png]({{site.baseurl}}/assets/images/stack5.png)
![stack6.png]({{site.baseurl}}/assets/images/stack6.png)


_Stack visualisation for this. Clearly slightly more complicated than the previous two._
