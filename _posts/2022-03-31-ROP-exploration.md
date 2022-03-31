---
published: true
layout: post
---
Currently studying/reading up on ARM assembly, and moving on to exploits -- this week I've read up on stack overflows and some basic Return Oriented Programming (ROP), following [this](https://azeria-labs.com/return-oriented-programming-arm32/) great tutorial on azerialabs. I suppose partially due to my complete lack of knowledge of this entire field, actually getting the exploit to work on my end was a lot more difficult than I felt it should've been, especially after reaching the endpoint (kind of like a hard math problem which seems a lot easier in hindsight, but when you were completely lost when trying to solve originally)

To run/write ARM code, I've been using the [Azeria Labs VM](https://azeria-labs.com/lab-vm-2-0/) on VirtualBox (I believe the intented software was VMware, so I had a LOT of trouble trying to get this to run on VirtualBox, but finally succeeded in the end)

This post will specifically touch on Challenge2 in the very first link, which comes included in the VM. My thought processes will probably assume one has, at the very least, some understanding of ARM syntax and the stack, since that is roughly what I had while trying to crack this. (For a decent runthrough of these concepts, do peruse the azerialabs website)

Another note is that the solution is highly Google-able as it is a very basic form of a ReturntoLibc attack. I suppose what I will document here are my mistakes and ideas that I entertained.

--------------------------------------------------

## **Buffer Overflows?**

So sometimes a program creates a buffer to contain a certain input from the user. 


{% highight c %}
  void Func1(char* input) {
      char buffer[80];
      strcpy(buffer, input);
  }
{% endhighlight %}

However, in a language like C, there isn't really a default system in place to prevent users from entering inputs **larger** than the buffer. So in the case of that, the excess input will **overflow** into other parts of memory, and it turns out that it is possible to control the content of what exaxctly overflows to those parts of memory.

Borrowing this image from AzeriaLab's writeup on this,
![stackoverflow.png]({{site.baseurl}}/_posts/stackoverflow.png)


The buffer content can actually overflow into the stack, which, among other things, is the place which saves the return address for a program when it branches into a function. Hence, by rewriting this return address into something else, we can get the program to just do something completely different after completing the function. Do read the [aforementioned post](https://azeria-labs.com/stack-overflow-arm32/) for greater detail.



--------------------------------------------------

## **What next?**

The goal in both challenges is to exploit the C file, and gain access to the [shell](https://en.wikipedia.org/wiki/Shell_(computing)).
![Screenshot 2022-03-30 at 8.45.20 PM.png]({{site.baseurl}}/_posts/Screenshot 2022-03-30 at 8.45.20 PM.png)


The solution to the first one was to overflow the buffer, and rewrite the PC to point to the stack, and from there execute shellcode (in this case, code that we have manually inserted to run the command "system(/bin/sh)" and gain access to the shell)

Now, this was all possible because the stack in Challenge1 had RWX permissions, ie Read, Write and Execute permisions.

In Challenge2, the XN (Execute Never) bit was turned on, and it is not longer as straightforward to just straight up run whatever code the attacker wants to inject. What we now want to do is to kind of piece together puzzle pieces, using code snippets that are already present in the system, in order to carry out what we want. Specifically here, we will be using code snippets from the libc (basically the preinstalled C library) in order to carry out our evil deeds, since libc has execute permissions.

![The stack only has read and write permissions.]({{site.baseurl}}/_posts/permissions.png)
_The stack only has read/write permissions, whereas part of libc does have execute permissions._


(While it seems dubious to hope that the code snippets you find in a system/machine can actually be pieced together to do what an attacker wants, it has been proven that ROP is Turing-Complete (that as a whole, the code snippets you get basically function like any other programming language out there))

Now, what we have to do first is to determine the **offset** of the program, ie how much it takes to overflow the buffer.

Fortunately, we can use our debugging tool (GDB) to figure this out instead of trial and error of running a massive number of As into the program.

![conveniently getting a massive output that we can paste into the program]({{site.baseurl}}/_posts/offset1.png)
_Generating a long string to crash the program with_

After this, we run the program through and observe the segfault that occurs, and do the following to find the offset:

![Running pattern search to determine at what point stuff overflows into the PC.]({{site.baseurl}}/_posts/offset2.png)
_Running pattern search to determine at exactly what point stuff overflows into the PC (Or rather what part of the code is popped into PC from the stack)_

(I am yet to fully grasp why we search for $pc + 1, but roughly I believe it is because in ARM the PC always adjusts itself for alignment purposes)



Anyways, we have discovered that the offset is 132, ie our 'payload' will begin with 132 characters of garbage before getting to the actual meat.

So, we have the beginnings of our exploit code:
![python1.png]({{site.baseurl}}/_posts/python1.png)

-------------------------------------------------------------
## **The actual ROPping**

Now, we need to figure out what code snippets to use in order to make the machine run system("/bin/sh").

Roughly, our final goal is:


**1) Have the PC pointing to the 'system' function, wherever it is in memory.**


**2) Have r0 contain the string "/bin/sh" (which, yes, is present in libc)**
_(why r0 specifically? Because the system function runs whatever is in r0 as the argument)_
  

That actually doesn't seem that hard! Let's begin with



### **Where are system and "/bin/sh"?**


![system.png]({{site.baseurl}}/_posts/system.png)
_It's as simple as that._


By running the code in the debugger, setting a breakpoint, and using the above command, we find that the address of system is at **0xb6f0b48c**.


![binsh.png]({{site.baseurl}}/_posts/binsh.png)


Next, we run this command (googleable) and we find that the string "/bin/sh" is stored in **0xb6fb407c**.


### **How do we control the registers?**


Now for the hard part. How do we get r0 and PC to point to what we require?


For r0, the **LDR** function comes to mind. We can try some sort of "**LDR r0, [SP]**" gadget if that exists (gadget = code snippet)


Time for our good friend Ropper. Cd-ing into the directory where libc resides, we run Ropper and use the command "**search /1/ LDR r0**" to search for any gadgets of length 1 (excluding the return instruction). _(Length 1 ensures that we minimise the number of instructions and thus the amount of complications we have to bother with)_


![ropper1.png]({{site.baseurl}}/_posts/ropper1.png)
![ropper2.png]({{site.baseurl}}/_posts/ropper2.png)
_The list is longer than this, but we are only interested in loading something from what SP is pointing to onto r0, not data from other registers._


Damn, looks like we aren't in luck. There is no **LDR r0, [SP]**. But isn't the **LDR r0, [SP, #4]** gadget the same thing? I just have to make "/bin/sh" appear 4 bytes further down. _Voila_, I thought, _this is it_.


At that time, my idea was:









