---
published: false
---
Currently studying/reading up on ARM assembly, and moving on to exploits -- this week I've read up on stack overflows and some basic Return Oriented Programming (ROP), following [this](https://azeria-labs.com/return-oriented-programming-arm32/) great tutorial on azerialabs. I suppose partially due to my complete lack of knowledge of this entire field, actually getting the exploit to work on my end was a lot more difficult than I felt it should've been, especially after reaching the endpoint (kind of like a hard math problem which seems a lot easier in hindsight, but when you were completely lost when trying to solve originally)

To run/write ARM code, I've been using the [Azeria Labs VM](https://azeria-labs.com/lab-vm-2-0/) on VirtualBox (I believe the intented software was VMware, so I had a LOT of trouble trying to get this to run on VirtualBox, but finally succeeded in the end)

This post will specifically touch on Challenge2 in the very first link, which comes included in the VM. My thought processes will probably assume one has, at the very least, some understanding of ARM syntax and buffer overflows, since that is roughly what I had while trying to crack this. (For a decent runthrough of these concepts, do peruse the azerialabs website)

Another note is that the solution is highly Google-able as it is a very basic form of a ReturntoLibc attack. I suppose what I will document here are my mistakes and ideas that I entertained.

--------------------------------------------------

**A basic explanation of a buffer overflow, I suppose**

So sometimes a program creates a buffer to contain a certain input from the user. 
{% highight C %}
void Func1(char* input) {
	char buffer[80];
    strcpy(buffer, input);
}
{% endhighlight %}

However, in a language like C, there isn't really a default system in place to prevent users from entering inputs **larger** than the buffer. So in the case of that, the excess input will **overflow** into other parts of memory, and it turns out that it is possible to control the content of what exaxctly overflows to those parts of memory.

Borrowing this image from AzeriaLab's writeup on this,
![]({{site.baseurl}}/https://azeria-labs.com/wp-content/uploads/2020/03/stack_2-3_darkbg-1.png)

The buffer content can actually overflow into the stack, which, among other things, is the place which saves the return address for a program when it branches into a function. Hence, by rewriting this return address into something else, we can get the program to just do something completely different after completing the function. Do read the [aforementioned post](https://azeria-labs.com/stack-overflow-arm32/) for greater detail.



--------------------------------------------------

**What next?**

The goal in both challenges is to exploit the C file, and gain access to the [shell](https://en.wikipedia.org/wiki/Shell_(computing)).
![Screenshot 2022-03-30 at 8.45.20 PM.png]({{site.baseurl}}/_posts/Screenshot 2022-03-30 at 8.45.20 PM.png)


The solution to the first one was to overflow the buffer, and rewrite the PC to point to the stack, and from there execute shellcode (in this case, code that we have manually inserted to run the command "system(/bin/sh)" and gain access to the shell)


