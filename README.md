Download Link: https://assignmentchef.com/product/solved-lab-3-buffer-overflows
<br>
<h2><a name="_Toc9966"></a>Overview</h2>

This assignment helps you develop a detailed understanding of the calling stack organization on an x86-64 processor. It involves applying a series of buffer overflow attacks on an executable file called <sub>bufbomb</sub>. (For some reason the textbook authors have a penchant for pyrotechnics.)

In this lab, you will gain firsthand experience with one of the methods commonly used to exploit security weaknesses in operating systems and network servers. Our purpose is to help you learn about the runtime operation of programs and to understand the nature of this form of security weakness so that you can avoid it when you write system code. We do not condone the use of these or any other form of attack to gain unauthorized access to any system resources. There are criminal statutes governing such activities.

<h2><a name="_Toc9967"></a>Instructions</h2>

<strong>Perform an update to your course-materials directory </strong>on the VM by running the update-course command from a terminal window. On the script’s success, you should find the provided code for lab3 in your course-materials directory. As a convenience, here is an archive of the course-materials directory as of this lab assignment: <a href="https://spark-public.s3.amazonaws.com/hardware/labs/lab3.tar.gz">lab3.tar.gz</a><a href="https://spark-public.s3.amazonaws.com/hardware/labs/lab3.tar.gz">.</a> The lab3 directory should then contain the following files:

<table width="664">

 <tbody>

  <tr>

   <td width="82">File Name</td>

   <td width="581">Description</td>

  </tr>

  <tr>

   <td width="82">makecookie</td>

   <td width="581">Generates a “cookie” based on some string (which will be your userid given in your <a href="https://class.coursera.org/hwswinterface-001/assignment/part_results?part_id=12">Lab0 feedback</a><a href="https://class.coursera.org/hwswinterface-001/assignment/part_results?part_id=12">)</a>.</td>

  </tr>

  <tr>

   <td width="82">bufbomb</td>

   <td width="581">The executable you will attack.</td>

  </tr>

  <tr>

   <td width="82">bufbomb.c</td>

   <td width="581">The important bits of C code used to compile <sub>bufbomb</sub>.</td>

  </tr>

  <tr>

   <td width="82">sendstring</td>

   <td width="581">A utility to help convert between string formats.</td>

  </tr>

  <tr>

   <td width="82">Makefile</td>

   <td width="581">For getting your exploits ready for submission.</td>

  </tr>

  <tr>

   <td width="82">USERID</td>

   <td width="581">File where you should place your Coursera userid from your <a href="https://class.coursera.org/hwswinterface-001/assignment/part_results?part_id=12">Lab0 feedback</a><a href="https://class.coursera.org/hwswinterface-001/assignment/part_results?part_id=12">.</a></td>

  </tr>

 </tbody>

</table>

1

All of these programs are compiled to run on the provided VM. The rest of the instructions assume that you will be performing your work on the VM, and you should make sure your solutions work on the before submitting them!

<strong>Be sure to read this write-up in its entirety before beginning your work.</strong>

<h3><a name="_Toc9968"></a>A Note about Line Endings</h3>

Linux (and UNIX machines in general) use a different line ending from Windows and traditional MacOS in text files. The reason for this difference is historical, early printers need more time to move the print head back to the beginning of the next line that to print a single character, so someone introduced the idea of separate line feed <sub>
 </sub>and carriage return <sub>r </sub>characters. Windows and HTTP use the <sub>r
 </sub>pairs, MacOS uses <sub>r </sub>and Linux uses <sub>
</sub>. In this lab it is important that your lines end with line feed (<sub>
</sub>) not any of the alternative line endings. If you are working on the VM or another Linux system this will probably not be a problem, but if you working across systems check your line endings.

<h3><a name="_Toc9969"></a>Making Your Cookie</h3>

A cookie is a string of eight bytes (or 16 hexadecimal digits) that is (with high probability) unique to you. You can generate your cookie with the <sub>makecookie </sub>program giving your Coursera userid (this can be found in your <a href="https://class.coursera.org/hwswinterface-001/assignment/part_results?part_id=12">Lab0 feedback</a> on <a href="https://class.coursera.org/hwswinterface-001/assignment/index">the assignments list page</a><a href="https://class.coursera.org/hwswinterface-001/assignment/index">)</a> as the argument:

$ ./makecookie your_coursera_userid

0x5e57e63274f39587

As an example, if your Coursera userid is 1397622, you would run ./makecookie 1397622

In most of the attacks in this lab, your objective will be to make your cookie show up in places where it ordinarily would not.

<h3><a name="_Toc9970"></a>Using the bufbomb Program</h3>

The <sub>bufbomb </sub>program reads a string from standard input with the function <sub>getbuf()</sub>:

unsigned long long getbuf() { char buf[36]; volatile char* variable_length; int i; unsigned long long val = (unsigned long long)Gets(buf); variable_length = alloca((val % 40) &lt; 36 ? 36 : val % 40); for(i = 0; i &lt; 36; i++) {

variable_length[i] = buf[i];

} return val % 40;

}

Don’t worry about what’s going on with <sub>variable_length </sub>and <sub>val </sub>and <sub>alloca() </sub>for now; all you need to know is that <sub>getbuf() </sub>calls the function <sub>Gets() </sub>and returns some arbitrary value.

The function <sub>Gets() </sub>is similar to the standard C library function <sub>gets()</sub>—it reads a string from standard input (terminated by ’<sub>
</sub>’) and stores it (along with a null terminator) at the specified destination. In the above code, the destination is an array <sub>buf </sub>having sufficient space for 36 characters.

Neither <sub>Gets() </sub>nor <sub>gets() </sub>has any way to determine whether there is enough space at the destination to store the entire string. Instead, they simply copy the entire string, possibly overrunning the bounds of the storage allocated at the destination.

If the string typed by the user to <sub>getbuf() </sub>is no more than 36 characters long, it is clear that <sub>getbuf() </sub>will return some value less than 0x28, as shown by the following execution example:

$ ./bufbomb

Type string: howdy doody

Dud: getbuf returned 0x20

It’s possible that the value returned might differ for you, since the returned value is derived from the location on the stack that <sub>Gets() </sub>is writing to. The returned value will also be different depending on whether you run the bomb inside gdb or run it outside of gdb for the same reason. Typically an error occurs if we type a longer string:

$ ./bufbomb

Type string: This string is too long and it starts overwriting things. Ouch!: You caused a segmentation fault!

As the error message indicates, overrunning the buffer typically causes the program state (e.g., the return addresses and other data structured that were stored on the stack) to be corrupted, leading to a memory access error. Your task is to be more clever with the strings you feed <sub>bufbomb </sub>so that it does more interesting things. These are called exploit strings. bufbomb must be run with the <sub>-u your_coursera_userid </sub>flag, which operates the bomb for the indicated Coursera user. (We will feed bufbomb your Coursera userid with the <sub>-u </sub>flag when grading your solutions.) bufbomb determines the cookie you will be using based on this flag value, just as the program <sub>makecookie </sub>does. Some of the key stack addresses you will need to use depend on your cookie.

<h3><a name="_Toc9971"></a>Formatting Your Exploit Strings</h3>

Your exploit strings will typically contain byte values that do not correspond to the ASCII values for printing characters. The program <sub>sendstring </sub>can help you generate these raw strings. <sub>sendstring </sub>takes as input a hex-formatted string and prints the raw string to standard output. In a hex-formatted string, each byte value is represented by two hex digits. Byte values are separated by spaces. For example, the string <sub>“012345” </sub>could be entered in hex format as <sub>30 31 32 33 34 35</sub>. (The ASCII code for decimal digit <sub>Z </sub>is <sub>0x3Z</sub>. Run man ascii for a full table.) Non-hex digit characters are ignored, including the blanks in the example shown. If you generate a hex-formatted exploit string in a file named <sub>exploit.txt</sub>, you can send it to <sub>bufbomb </sub>through a couple of <a href="http://www.westwind.com/reference/os-x/commandline/pipes.html">unix pipes</a><a href="http://www.westwind.com/reference/os-x/commandline/pipes.html">:</a>

$ cat exploit.txt | ./sendstring | ./bufbomb -u your_coursera_userid Or you can store the raw bytes in a file and use I/O redirection to supply it to <sub>bufbomb</sub>:

$ ./sendstring &lt; exploit.txt &gt; exploit.bytes

$ ./bufbomb -u your_coursera_userid &lt; exploit.bytes

With the above method, when running <sub>bufbomb </sub>from within <sub>gdb</sub>, you can pass in the exploit string as follows:

$ gdb ./bufbomb

(gdb) run -u your_coursera_userid &lt; exploit.bytes

One important point: your exploit string must not contain byte value <sub>0x0A </sub>at any intermediate position, since these are the ASCII codes for newline (’<sub>
</sub>’). When <sub>Gets() </sub>encounters either of these byte, it will assume you intended to terminate the string. <sub>sendstring </sub>will warn you if it encounters this byte value.

When using <sub>gdb</sub>, you may find it useful to save a series of <sub>gdb </sub>commands to a text file and then use the <sub>-x </sub>commands.txt flag. This saves you the trouble of retyping the commands every time you run <sub>gdb</sub>. You can read more about the <sub>-x </sub>flag in <sub>gdb</sub>’s <sub>man </sub>page.

<h3><a name="_Toc9972"></a>Generating Byte Codes</h3>

You may wish to come back and read this section later after looking at the problems.

Using <sub>gcc </sub>as an assembler and <sub>objdump </sub>as a disassembler makes it convenient to generate the byte codes for instruction sequences. For example, suppose we write a file <sub>example.s </sub>containing the following assembly code:

# Example of hand-generated assembly code movq $0x1234abcd,%rax # Move 0x1234abcd to %rax pushq $0x401080         # Push 0x401080 on to the stack retq               # Return

The code can contain a mixture of instructions and data. Anything to the right of a ’<sub>#</sub>’ character is a comment.

We can now assemble and disassemble this file:

$ gcc -c example.s

$ objdump -d example.o &gt; example.d

The generated file <sub>example.d </sub>contains the following lines

0:             48 c7 c0 cd ab 34 12              mov           $0x1234abcd,%rax

7:    68 80 10 40 00     pushq $0x401080 c:           c3            retq

Each line shows a single instruction. The number on the left indicates the starting address (starting with 0), while the hex digits after the ’<sub>:</sub>’ character indicate the byte codes for the instruction. Thus, we can see that the instruction pushq $0x401080 has a hex-formatted byte code of 68 80 10 40 00.

If we read off the 4 bytes starting at address 8 we get: <sub>80 10 40 00</sub>. This is a byte-reversed version of the data word <sub>0x00401080</sub>. This byte reversal represents the proper way to supply the bytes as a string, since a little-endian machine lists the least significant byte first.

Finally, we can read off the byte sequence for our code (omitting the final <sub>0</sub>’s) as:

48 c7 c0 cd ab 34 12 68 80 10 40 00 c3

<h2><a name="_Toc9973"></a>The Exploits</h2>

There are three functions that you must exploit for this lab. The exploits increase in difficulty. For those of you looking for a challenge, there is a fourth function you can exploit for extra credit.

<h3><a name="_Toc9974"></a>Level 0: Candle</h3>

The function getbuf() is called within bufbomb by a function test():

void test()

{ volatile unsigned long long val; volatile unsigned long long local = 0xdeadbeef; char* variable_length;

entry_check(3); /* Make sure entered this function properly */ val = getbuf();

if (val &lt;= 40) {

variable_length = alloca(val);

} entry_check(3);

/* Check for corrupted stack */ if (local != 0xdeadbeef) { printf(“Sabotaged!: the stack has been corrupted
”);

} else if (val == cookie) { printf(“Boom!: getbuf returned 0x%llx
”, val); if (local != 0xdeadbeef) { printf(“Sabotaged!: the stack has been corrupted
”);

} if (val != cookie) { printf(“Sabotaged!: control flow has been disrupted
”);

} validate(3);

} else { printf(“Dud: getbuf returned 0x%llx
”, val); }

}

When <sub>getbuf() </sub>executes its return statement, the program ordinarily resumes execution within function test(). Within the file <sub>bufbomb</sub>, there is a function <sub>smoke()</sub>:

void smoke()

{ entry_check(0); /* Make sure entered this function properly */ printf(“Smoke!: You called smoke()
”); validate(0); exit(0);

}

Your task is to get <sub>bufbomb </sub>to execute the code for <sub>smoke() </sub>when <sub>getbuf() </sub>executes its return statement, rather than returning to <sub>test()</sub>. You can do this by supplying an exploit string that overwrites the stored return pointer in the stack frame for <sub>getbuf() </sub>with the address of the first instruction in <sub>smoke</sub>. Note that your exploit string may also corrupt other parts of the stack state, but this will not cause a problem, because smoke() causes the program to exit directly.

<strong>Advice:</strong>

<ul>

 <li>All the information you need to devise your exploit string for this level can be determined by examining a disassembled version of <sub>bufbomb</sub>.</li>

 <li>Be careful about byte ordering.</li>

 <li>You might want to use <sub>gdb </sub>to step the program through the last few instructions of <sub>getbuf() </sub>to make sure it is doing the right thing.</li>

 <li>The placement of <sub>buf </sub>within the stack frame for <sub>getbuf() </sub>depends on which version of <sub>gcc </sub>was used to compile <sub>bufbomb</sub>. You will need to pad the beginning of your exploit string with the proper number of bytes to overwrite the return pointer. The values of these bytes can be arbitrary.</li>

 <li>Check the line endings if your smoke.txt with <sub>hexdump -C smoke.txt</sub>.</li>

</ul>

<h3><a name="_Toc9975"></a>Level 1: Sparkler</h3>

Within the file <sub>bufbomb </sub>there is also a function <sub>fizz()</sub>:

void fizz(int arg1, char arg2, long arg3, char* arg4, short arg5, short arg6, unsigned long long val)

{ entry_check(1); /* Make sure entered this function properly */ if (val == cookie)

{ printf(“Fizz!: You called fizz(0x%llx)
”, val); validate(1);

} else { printf(“Misfire: You called fizz(0x%llx)
”, val);

} exit(0);

}

Similar to Level 0, your task is to get <sub>bufbomb </sub>to execute the code for <sub>fizz() </sub>rather than returning to <sub>test</sub>. In this case, however, you must make it appear to <sub>fizz </sub>as if you have passed your cookie as its argument. You can do this by encoding your cookie in the appropriate place within your exploit string.

<strong>Advice:</strong>

<ul>

 <li>Note that in x86-64, the first six arguments are passed into registers and additional arguments are passed through the stack. Your exploit code needs to write to the appropriate place within the stack.</li>

 <li>You can use <sub>gdb </sub>to get the information you need to construct your exploit string. Set a breakpoint within getbuf() and run to this breakpoint. Determine parameters such as the address of <sub>global_value </sub>and the location of the buffer.</li>

</ul>

<h3><a name="_Toc9976"></a>Level 2: Firecracker</h3>

A much more sophisticated form of buffer attack involves supplying a string that encodes actual machine instructions. The exploit string then overwrites the return pointer with the starting address of these instructions. When the calling function (in this case <sub>getbuf</sub>) executes its <sub>ret </sub>instruction, the program will start executing the instructions on the stack rather than returning. With this form of attack, you can get the program to do almost anything. The code you place on the stack is called the exploit code. This style of attack is tricky, though, because you must get machine code onto the stack and set the return pointer to the start of this code.

For level 2, you will need to run your exploit within <sub>gdb </sub>for it to succeed. (the VM has special memory protection that prevents execution of memory locations in the stack. Since <sub>gdb </sub>works a little differently, it

will allow the exploit to succeed.)

Within the file <sub>bufbomb </sub>there is a function <sub>bang()</sub>:

unsigned long long global_value = 0;

void bang(unsigned long long val)

{ entry_check(2); /* Make sure entered this function properly */ if (global_value == cookie)

{ printf(“Bang!: You set global_value to 0x%llx
”, global_value); validate(2);

} else { printf(“Misfire: global_value = 0x%llx
”, global_value);

} exit(0);

}

Similar to Levels 0 and 1, your task is to get <sub>bufbomb </sub>to execute the code for <sub>bang() </sub>rather than returning to <sub>test()</sub>. Before this, however, you must set global variable <sub>global_value </sub>to your cookie. Your exploit code should set <sub>global_value</sub>, push the address of <sub>bang() </sub>on the stack, and then execute a <sub>retq </sub>instruction to cause a jump to the code for <sub>bang()</sub>.

<strong>Advice:</strong>

<ul>

 <li>Determining the byte encoding of instruction sequences by hand is tedious and prone to errors. You can let tools do all of the work by writing an assembly code file containing the instructions and data you want to put on the stack. Assemble this file with <sub>gcc </sub>and disassemble it with <sub>objdump</sub>. You should be able to get the exact byte sequence that you will type at the prompt. (A brief example of how to do this is included in the Generating Byte Codes section above.)</li>

 <li>Keep in mind that your exploit string depends on your machine, your compiler, and even your cookie. Make sure your exploit string works on the VM, and make sure you include your Coursera userid on the command line to <sub>bufbomb</sub>.</li>

 <li>Watch your use of address modes when writing assembly code. Note that <sub>movq $0x4, %rax </sub>moves the value 0x0000000000000004 into register %rax; whereas movq 0x4, %rax moves the value <em>at </em>memory location <sub>0x0000000000000004 </sub>into <sub>%rax</sub>. Because that memory location is usually undefined, the second instruction will cause a segmentation fault!</li>

 <li>Do not attempt to use either a <sub>jmp </sub>or a <sub>call </sub>instruction to jump to the code for <sub>bang()</sub>. These instructions use PC-relative addressing, which is very tricky to set up correctly. Instead, push an address on the stack and use the <sub>retq </sub></li>

</ul>

<h3><a name="_Toc9977"></a>Extra Credit – Level 3: Dynamite</h3>

For level 3, you will need to run your exploit within <sub>gdb </sub>for it to succeed.

Our preceding attacks have all caused the program to jump to the code for some other function, which then causes the program to exit. As a result, it was acceptable to use exploit strings that corrupt the stack, overwriting the saved value of register <sub>%rbp </sub>and the return pointer.

The most sophisticated form of buffer overflow attack causes the program to execute some exploit code that patches up the stack and makes the program return to the original calling function (<sub>test() </sub>in this case). The calling function is oblivious to the attack. This style of attack is tricky, though, since you must: (1) get machine code onto the stack, (2) set the return pointer to the start of this code, and (3) undo the corruptions made to the stack state.

Your job for this level is to supply an exploit string that will cause <sub>getbuf() </sub>to return your cookie back to test(), rather than the value 1. You can see in the code for <sub>test() </sub>that this will cause the program to go “<sub>Boom!</sub>“. Your exploit code should set your cookie as the return value, restore any corrupted state, push the correct return location on the stack, and execute a <sub>ret </sub>instruction to really return to <sub>test()</sub>.

<strong>Advice:</strong>

<ul>

 <li>In order to overwrite the return pointer, you must also overwrite the saved value of <sub>%rbp</sub>. However, it is important that this value is correctly restored before you return to <sub>test()</sub>. You can do this by either (1) making sure that your exploit string contains the correct value of the saved <sub>%rbp </sub>in the correct position, so that it never gets corrupted, or (2) restore the correct value as part of your exploit code.</li>

</ul>

You’ll see that the code for <sub>test() </sub>has some explicit tests to check for a corrupted stack.

<ul>

 <li>You can use <sub>gdb </sub>to get the information you need to construct your exploit string. Set a breakpoint within <sub>getbuf() </sub>and run to this breakpoint. Determine parameters such as the saved return address and the saved value of <sub>%rbp</sub>.</li>

 <li>Again, let tools such as <sub>gcc </sub>and <sub>objdump </sub>do all of the work of generating a byte encoding of the instructions.</li>

 <li>Keep in mind that your exploit string depends on your machine, your compiler, and even your cookie. Make sure your exploit string works on the VM, and make sure you include your Coursera userid on the command line to <sub>bufbomb</sub>.</li>

</ul>

Reflect on what you have accomplished. You caused a program to execute machine code of your own design. You have done so in a sufficiently stealthy way that the program did not realize that anything was amiss.

execve is system call that replaces the currently running program with another program inheriting all the open file descriptors. What are the limitations the exploits you have preformed so far? How could calling execve allow you to circumvent this limitation? If you have time, try writing an additional exploit that uses execve and another program to print a message.