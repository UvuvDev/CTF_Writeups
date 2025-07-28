
# DeadSec 2025 - Rev - DeadFib

The binary (DeadFib) is in the same directory as this writeup.

First thing's first: check the file type. 

```
DeadFib: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c04ea441f99bda460d256f907d9ca669bd26da7e, for GNU/Linux 3.2.0, with debug_info, not stripped
```

It's an ELF with some debug symbols, so let's pop it in Ghidra to decompile. We open it up and see that it is obfuscated Rust code. 

<img width="794" height="584" alt="image" src="https://github.com/user-attachments/assets/fe6eb59d-cb24-4a8c-86ed-84aa0a14e169" />


Looking at this is quite overwhelming if you haven't touched Rust rev before. Number one piece of advice when doing reverse engineering with Rust: the library calls are often disconnected from how we normally write code.

Line 197 in main() is

```rust
::alloc::string::String::new(&local_410);
```

In this decompilation, Ghidra does a variable decleration by calling the variable as a parameter to a class function. This is very common in decompiled OOP languages, and something to get used to as you reverse engineer this. Understanding the disjointedness of Rust like in lines 199 and 201 where there's not even a parameter to the stdin() and readline() calls will make your time much easier.

Now to actually reverse this. local_410 is clearly the user_input string. We see in lines 207-209 that it does operation's to trim the string's excess off (&var4 is the trimmed string, renamed it trimmed_str). We see this big if statement, and looking at the conditional, uVar2 is the string length of the trimmed user input. So the user can input 0x29 (or 41) bytes of input (not including suffixed spaces). If they don't, nothing interesting happens. This tells us the flag is certainly 41 characters long, which is *important* for later.

Next, let's do some dynamic analysis and see what this program actually does. Running it prompts us with an input (I put 41 A's) and then it let's us know output.bin was written to, and then it gives us a hash and some other data.

<img width="629" height="213" alt="image" src="https://github.com/user-attachments/assets/6124f2cd-947b-4573-baa9-33bf2deacea2" />

I was thrown off by the hash for a while thinking it was important. But, when you think about it, the only thing we were given is the output.bin of the flag. There was no hash, avalanche coefficient, or data entropy that was given to us to help us. If we go through the program (I'll spare you the time I spent), you see that there is only one string written to output.bin (this is on line 383 - std::fs::File::write_all<>((File*)&local_ec, &Var5) ): so let's backtrack from there.

local_250 is only used in one place before being written (hint: use Secondary Highlight!), lines 287 - 311 (and it's variable definition on line 278).

<img width="789" height="660" alt="image" src="https://github.com/user-attachments/assets/6c27dac9-e2e7-4502-af76-b02ddb92e8c1" />

It's made, then the user input (after some variable redefinitions earlier) get's broken into byte-sized pieces and iterated through (think for each loop!). then it's put into 2 functions: wP8nL3kX6() and yS4nF7kM1(). Cool. Now we have to reverse these!

Looking at the source code, I genuinely do not know what is going on. And that's okay! An easy way to do this is just set a breakpoint before and after the function, and see what happens. The definition is this:

```rust
String * __rustcall
fib_challenge::fib_challenge::wP8nL3kX6(String *__return_storage_ptr__,u64 xQ2nR9fT5)
```

*_return_storage_ptr__* is the parameter we'll care about, since xQ2nR9fT5 is never written to anyways. Doing this reveals that it get's a number and outputs it's binary format. If you've looked at the given output.bin, it's all 1's and 0's, so this makes sense.

```rust
String * __rustcall
fib_challenge::fib_challenge::yS4nF7kM1(String *__return_storage_ptr__,&str zL9nT3qX8)

{
  Chars CVar1;
  Map<> self;
  
  CVar1 = core::str::chars(zL9nT3qX8);
  self.iter.iter =
       (Chars)core::iter::traits::iterator::Iterator::map<>
                        (CVar1.iter.ptr.pointer,CVar1.iter.end_or_len);
  core::iter::traits::iterator::Iterator::collect<>(__return_storage_ptr__,self);
  return __return_storage_ptr__;
}
```
yS4nF7kM1() is different: it uses a map() call to perform an action on the parameter (which is the array of chars inside the string zL9nT3qX8). It calls a function pointer that you can assign. I for the life of me could not find where this map function pointer is set. So I once again checked it's functionality dynamically, and looked at the stack / parameters.

We get this string inputted into our function, at this memory location: 
```
0x5555555de410: 0x3130303131313131      0x3031303031303131
0x5555555de420: 0x3131313031303031      0x3030303130313031
0x5555555de430: 0x3130303131303130      0x0000000031303131
```
Then afterwards, this string doesn't change, *but* looking 16 bytes down the stack we see a new string:
```
0x5555555de450: 0x3031313030303030      0x3130313130313030
0x5555555de460: 0x3030303130313130      0x3131313031303130
0x5555555de470: 0x3031313030313031      0x0000000030313030
```
And if you look carefully, every single 0x30 has been changed to 0x31 and vice versa. So this is just a not function, but for ascii string's of 1's and 0's.

With that done, we know what this loop is doing: get's some numbers, changes them to a string of ASCII 1's and 0's, and then makes them flip. It begs the question "which numbers?"

If you trace back, you'll find this function call on line 188: 
```rust
rW9nT2kF7(&local_4b8,500);
```
If you go in here and reverse it, you'll find it's the Fibonacci sequence up to 500 returned in an array. I just made some psuedocode and recognized it from running through some iterations of the function output. Your method may vary. 

So now we see the pipeline:

1. Fibonacci sequence gets generated
2. Get user input
3. Use the ASCII codes as indexes into this fibonacci sequence
4. Convert corresponding fibonacci number into a string of "1"s and "0"s
5. Flip these "1"s and "0"s
6. Repeat 3-5 41 times for each letter of the input
7. Write final string to output.bin

To verify this, we can check the first letter of the flag format ( DEAD{} ) and see if the bits match up with the Fibonacci number for 0x44.

Fib of 0x44: 10000100010010001000000000000111101001001001101
output.bin:  01111011101101110111111111111000010110110110010

Looks all good. I wrote a quick Python command script to flip every 1 to a 0 and vice versa and print it so I could paste it directly into my new script. 

My first thought was to try to Z3 this, but it doesn't really work here. The constraints are 6 known characters ( DEAD{ as a prefix, } as a suffix), and a total length of 1885 bits. But those bits can be arranged in any way. If you think about how it's encoded, it can only be a fibonacci number between 0x20 and 0x7e. So... why not check until you find a matching number? The Fibonacci numbers are dozens of bits long, there's no chance that they'll be identical bitstrings inside of others (say, Fn = 0x20 being inside of Fn = 0x7b). So my script just runs through the whole thing, checks if there's a Fib of N number matching, and if there is, print out the ASCII for N.   

Script:

```python

flagData = "1000010001001000100000000000011110100100100110111010110000010011110100110000101111111001000010111110011101001010010111101010000101100111011000010001001000100000000000011110100100100110110100101010110100011000010110101111111001100001101000000100001010001011110000110100110000111100101011000100100111110001000100011110101100000011001010000111100011100010000111100111111010111101011101000001100010001100100100000110101010001010110100100000110011010001010000001111111110110001011011010101101100101111011101100101101000100000101111010110000001100101000011110001110001011110000110100110000111100101011000100100111110001000100010001011110000110100110000111100101011000100100111110001000100011010010010101110010110110001011110101100010000111100111111010111101011101000001100010001100100100000111000110100111100110000010100110010111001111011011011101101010011111000101100101001011111111000011100010111100001101001100001111001010110001001001111100010001000111001111101001100010111100100001110011110110110111011010100111110001011001010010111111110000111100111101101101110110101001111100010110010100101111111100001111110101100000011001010000111100011100000011111111101100010110110101011011000101111000011010011000011110010101100010010011111000100010001100011010011110011000001010011001011101111000101010101110001100111011001010111110110111100100000010110001101001111001100000101001100101101010100010101101001000001100110100010110111100010101010111000110011101100101011111011011110010000001011100111110100110001011110010000111010010010101110010110110001011110101100101111011101100101101000100000101110011111010011000101111001000011111010110000001100101000011110001110111011100011001100111001011000011000111101000110100001010010000001001011110111011001011010001000001010111011100011001100111001011000011110011111010011000101111001000011110110100001111010011011111111110001000101110100001010101110101"
MASK64 = (1 << 64) - 1

fibSeq = {}
def fibonacci(n, memo={}):
    if n <= 0:
        return 0
    elif n == 1:
        return 1
    elif n in memo:
        return memo[n]
    else:
        memo[n] = (fibonacci(n-1) + fibonacci(n-2)) & MASK64
        return memo[n]

# Make the dictionary have up to the ascii value
fibonacci(0x7e, fibSeq)

left = 0 

print("fibonacci calcuated. beginning flag calculation\n")

while left < len(flagData):
/
    L = 0

    for i in range(0x20, 0x7f):
        fib = fibonacci(i)

        L = fib.bit_length()

        if left + L > len(flagData):
            print("thats all.")
            exit()

        if fibonacci(i) == int(flagData[left:(left+L)], 2):
            print(chr(i), end = "")
            left += L
            break


    #print(left, end = " ")

    

print("")
print("Exit!")

```

<img width="759" height="115" alt="image" src="https://github.com/user-attachments/assets/adb883df-a133-49ae-98e9-5bbd7006da47" />

