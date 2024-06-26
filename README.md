# Binary Golf 5 - Linux shellcoding ideas
It is June 2024, and [Binary Golf](https://binary.golf/) season is upon us!  
For the uninitiated - the idea of Binary Golf is to create the shortest program\script\whatever that does *some task*, where the task changes each time.  
This is [year 5](https://binary.golf/5/), and the goal is:

```
Create the smallest file that downloads this text file and displays its contents.
```
(The `this` part is really [https://binary.golf/5/5](https://binary.golf/5/5)).

There are some interesting clarifying questions that were kind of answered:
- It's okay to rely on external dependencies of the OS.
- Environment variables or commandline arguments are okay, although they might be in different categories.
- The term "download" means download to memory also - you do not have to touch disk.

## Preliminary thoughts
Some ideas that come to mind:
1. The URL is `https`, and implementing your own TLS library (or borrowing one) would increase solution size drastically. Therefore, relying on external binaries or libraries makes sense.
2. The `curl` binary exists in every primary modern OS (Windows, Linux, macOS).
3. The URL path could be shortened easily (e.g. like using `bit.ly`).
4. We could send the URL in a commandline argument (or environment variable), or even an entire program!

To test some of those ideas (some of them "game the system" by definition) I submitted [one exterme solution](https://github.com/binarygolf/BGGP/blob/main/2024/entries/jbo/jbo.sh.txt) with only 2 bytes:

```shell
$1
```

To run: `bash jbo.sh "curl https://binary.golf/5/5"`.

In my opinion, this is definitely cheating, but whatever. My real goal is not to rely on scripting but to do some binary work, and so I've decided to go for a shellcode!

## Shellcode ideas
For now I've decided to perform a Linux shellcode, since they're much easier. Also, I really wanted to rely on `curl`.  
I already found one shortened URL that someone submitted, so I'll just use it: `7f.uk`.  
Now, two ideas come to mind for calling `curl`:
1. Find the `system` API in `libc` in some way and run `curl -L 7f.uk`.
2. Call `execve` using a `syscall`.

### Finding libc!system using TSX-based egg-hunting
Normally, before things like [ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization), you could rely on the `libc` library to be loaded at a constant address.  
These days `libc` is going to be loaded at a random address, so we have to find it (just like in a real exploit). How does one find `libc` using a shellcode?  
Also, let's keep in mind different `libc` versions might have the `system` symbol live in different offsets, and I wanted to be as generic as possible.  
Well, I've decided to borrow an idea from [Dan Clemente's blogpost on TSX-based egg-hunting](https://bugnotfound.com/posts/htb-business-ctf-2024-abusing-intel-tsx-to-solve-a-sandbox-challenge/).  
Dan has done an awesome work during [HTB Business CTF 2024](https://ctf.hackthebox.com/event/details/htb-business-ctf-2024-the-vault-of-hope-1474) but here's the gist of it:
- Intel [TSX](https://en.wikipedia.org/wiki/Transactional_Synchronization_Extensions) is an extension to the Intel ISA that supports transactional memory.
- The idea is to find a "pattern" in the entire memory by capturing bad memory access using the `xbegin` and `xend` instructions.

So, the first question I had was whether there's a unique pattern in `libc!system` that I could find. Well, as it turns out, `4 bytes` were enough to find it!  
I've done the following:
1. Using the `nm` binary I've listed the symbols in `libc` to find the offset of `system`.
2. I opened the `libc.so.6` library and read all its bytes, and then by brute-force I looked for a unique pattern - one which only exists in `system`.
3. I decided to look for the unique pattern only until a `RET` (0xC3) instruction, even though it's not entirely necessary.
4. I save the offset and the pattern.

As it turns out, the pattern `FF 74 07 E9` is found `6 bytes` inside `libc!system`. Here's some code that finds it:

```python
#!/usr/bin/env python3
import subprocess
import binascii

# Constants
NM_PATH = '/usr/bin/nm'
LIBC_PATH = '/usr/lib/x86_64-linux-gnu/libc.so.6'
SYSTEM_SYMBOL_PATTERN = 'system@@'
RET_INST = b'\xC3'
UNIQUE_PATTERN_LEN = 4

def get_system_pattern_with_offset():
    """
        Finds a unique pattern in libc!system API.
        Returns the tuple: (pattern, pattern_offset_from_system, system_offset_from_libc)
    """

    # Get the system symbol offset
    proc = subprocess.run([ NM_PATH, '-D', LIBC_PATH ], capture_output=True)
    system_offsets = set([ int(line.split(' ')[0], 16) for line in proc.stdout.decode().split('\n') if SYSTEM_SYMBOL_PATTERN in line ])
    if len(system_offsets) == 0:
        raise Exception('Cannot find libc offsets')
    if len(system_offsets) > 1:
        raise Exception('Ambiguity in libc offsets')
    system_offset = system_offsets.pop()
    print(f'[+] Found libc!system offset at 0x{system_offset:02x}')

    # Read libc bytes
    with open(LIBC_PATH, 'rb') as fp:
        libc_bytes = fp.read()

    # Get the system API bytes
    system_bytes = libc_bytes[system_offset:]
    first_ret_index = system_bytes.find(RET_INST)
    if first_ret_index < 0:
        raise Exception('Could not find RET instruction')
    system_bytes = system_bytes[:first_ret_index+1]

    # Find unique bytes in system
    found_pattern = False
    for i in range(len(system_bytes) - UNIQUE_PATTERN_LEN):
        pattern = system_bytes[i:i+UNIQUE_PATTERN_LEN]
        if libc_bytes.find(pattern) < system_offset:
            continue
        if libc_bytes.rfind(pattern) > system_offset + len(system_bytes):
            continue
        found_pattern = True
        break

    # Validate we found a pattern
    if not found_pattern:
        raise Exception('Could not find unique byte pattern')
    pattern_location = system_bytes.find(pattern)
    print(f'[+] Found unique pattern "{binascii.hexlify(pattern)}" {pattern_location} bytes into libc!system')

    # Return data
    return (pattern, pattern_location, system_offset)
```

So, my shellcode is quite simple, even though I saw one issue when calling `libc!system` - it saves MMX registers on the stack, which means we have to keep the stack 16-bytes aligned! Therefore:
1. My shellcode will align the stack to 16-bytes.
2. My shellcode will initialize `RSI` and look for the unique pattern (saved in `ECX`) between `xbegin` and `xend` - bad memory access will roll to the *relative* address mentioned in `xbegin`, which suits shellcodes well. Note that since Intel is [Little Endian](https://en.wikipedia.org/wiki/Endianness) the `ECX` value is `0xe90774ff`.
3. Once pattern is found - use the offset to point `RSI` to `libc!system`.
4. Prepare the sole argument in `RDI` by using `call-pop`.

So, here's my shellcode (I use [nasm](https://nasm.us/)):

```assembly
[BITS 64]

; Constants acquired from preparation
UNIQUE_PATTERN_BYTES EQU 0xe90774ff
SYSTEM_OFFSET_FROM_LIBC EQU 0x50d70
PATTERN_OFFSET_FROM_SYSTEM EQU 0x06

        ; Make some stack is 16-byte aligned
        and rsp, 0xFFFFFFFFFFFFFFF0

        ; Egg-hunting the unique bytes
        mov rsi, SYSTEM_OFFSET_FROM_LIBC + PATTERN_OFFSET_FROM_SYSTEM
        mov ecx, UNIQUE_PATTERN_BYTES

egg_hunt:
        add rsi, 0x1000
        xbegin egg_hunt
        cmp ecx, [rsi]
        xend
        jne egg_hunt

        ; Point to libc!system
        sub rsi, PATTERN_OFFSET_FROM_SYSTEM

        ; Push the commandline
        call call_system
        db 'curl -L 7f.uk', 0

call_system:
        pop rdi
        call rsi

; Hang
jmp $
```

The terms didn't mention whether the program should exist or not - for all means and purposes I could've let it crash, but I've decided to hang forver using `jmp $`.  
With that, I got a shellcode of `62 bytes`.  
Also, apparently `gdb` does not like debugging `TSX` - even doing `si` on `xbegin` sometimes acts as if we're doing an entire transaction iteration!  
Lastly, this takes *a lot of time* to run (like 15 minutes on a modern PC) due to exhausing the entire memory space (but we increase by `0x1000` which should be okay).  
One optimization we could've done is using the `cmpsd` or `scasd`, but it'd be even slower (since we move byte-by-byte) and only save `4` bytes in total (taking `cld` instruction into account).

### Shellcoding with syscall
On Linux, one can simply use the `syscall` numbers freely, which makes shellcoding much easier on Linux (I've already talked about Windows shellcoding [here](https://github.com/yo-yo-yo-jbo/msf_shellcode_analysis/)).  
Well, the [execve](https://man7.org/linux/man-pages/man2/execve.2.html) syscall is just around the corner! But we need to prepare `argv` for it:

```c
int execve(const char *pathname, char *const _Nullable argv[], char *const _Nullable envp[]);
```

The good news:
1. No need to locate anything in memory - the `execve` syscall number is well known (`59`).
2. No need to hang the program - `execve` shouldn't return in case of success.

The bad news:
1. `execve` should have the `pathname` be an absolute path - writing `curl` doesn't help - we need `/bin/curl`.
2. Preparing `argv` is necessary and means pointing to an array of pointers (since strings are pointers too).

Here's what I got with only `46` bytes:

```assembly
[BITS 64]

; Constants
EXECVE_SYSCALL_NUM EQU 0x3B

        ; Jump over argv
        call after_argv

        ; argv in reverse order
        db `7f.uk\x00`
        db `-L\x00`
        db `/bin/curl\x00`

after_argv:

        ; Save argv pointer
        pop rdi

        ; Set the syscall number into RAX
        push EXECVE_SYSCALL_NUM
        pop rax

        ; Set envp (from RAX to RDX)
        cdq

        ; Prepare argv in stack
        push rdx
        push rdi
        add rdi, 6
        push rdi
        add rdi, 3
        push rdi        ; At this point RDI also points to main executable
        mov rsi, rsp

        ; Call execve
        syscall
```

I think this is self-explanatory - the only "complicated" part is preparing `argv` on the stack.  
The x64 calling convernsion is:
- First argument is `rdi` which should point to the string `/bin/curl`.
- Second argument is `rsi` which points to our `argv`, which is composed to four pointers to the following: `{ "/bin/curl", "-L", "7f.uk", NULL }`.
- Third argument is `rdx` which points to `envp`. We set it to be `NULL` using the `cdq` trick.

The few minor tricks besides that:
1. I first set the `execve` syscall number in `rax`. I do that not only to make `rax` hold the syscall number, but also to prepare for the next instruction (`cdq`).
2. I use `cdq` to make `rdx` zero with less bytes - it sign extends `rax` to `rdx`.

### Running the shellcode
For completeness, here's C code that runs an arbitrary shellcode:

```c
#include <unistd.h>
#include <stdio.h>
#include <sys/mman.h>

#define UNMAP(ptr, size)        do                                      \
                                {                                       \
                                        if (NULL != (ptr))              \
                                        {                               \
                                                munmap((ptr), (size));  \
                                                (ptr) = NULL;           \
                                        }                               \
                                }                                       \
                                while (0)

#define FCLOSE(fp)              do                                      \
                                {                                       \
                                        if (NULL != (fp))               \
                                        {                               \
                                                fclose(fp);             \
                                                (fp) = NULL;            \
                                        }                               \
                                }                                       \
                                while (0)

static void run_shellcode(char* buffer)
{
        int (*exeshell)() = (int (*)())buffer;
        (int)(*exeshell)();
}

int main(int argc, char** argv)
{
        FILE* fp = NULL;
        long fsize = 0;
        char* buffer = NULL;
        int (*exeshell)() = NULL;

        // Check arguments
        if (2 != argc)
        {
                printf("Invalid number of arguments: %d", argc);
                goto cleanup;
        }

        // Open the file for reading
        fp = fopen(argv[1], "rb");
        if (NULL == fp)
        {
                printf("Error opening file for reading: %s", argv[1]);
                goto cleanup;
        }

        // Get the file size
        fseek(fp, 0, SEEK_END);
        fsize = ftell(fp);
        fseek(fp, 0, SEEK_SET);

        // Allocate shellcode buffer
        buffer = mmap(NULL, fsize, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        if (NULL == buffer)
        {
                printf("Error allocating RWX memory of size: %ld", fsize);
                goto cleanup;
        }

        // Read file and close it
        fread(buffer, fsize, 1, fp);
        FCLOSE(fp);

        // Run shellcode
        run_shellcode(buffer);

cleanup:

        // Free resources
        UNMAP(buffer, fsize);
        FCLOSE(fp);

        // Return result
        return 0;
}
```

## Creating a polyglot
For now, I've decided to create a Polyglot of shell script and the Linux shellcode I have.
The issue is that shells do not treat non-printable characters well - `bash` refuses to run binary files, but `sh` does a bit better as long as we quit before any non-printable characters occur.  
My plan is therefore to run `sh ./shellcode`, and having the shellcode prefix to be a variable:

```shell
VARIABLE_NAME=<binary junk>
curl <url>
exit
```

I could work with either newline seperators or the `;` character - and in fact, it just so happens that the syscall number to `execve` is exactly interpreter as `;`. Let us use that!  
But for now, I need to start with the variable name - legal shell variable names can only contain the underscore `_` symbol, as well as English letters (in either casing) and digits (as long as we do not start with a digit). We can also afford remarks (`#`) and whitespaces.  
Working by hand is exhausing - I ended up using [Capstone](https://www.capstone-engine.org) to disassemble printable characters and showing me what can be legal and non-destructive x64 assembly:

```python3
from capstone import *

def f(c):
    code = c.encode() * 10
    md = Cs(CS_ARCH_X86, CS_MODE_64)
    print(f'==={c}===')
    for i in md.disasm(code, 0x1000):
        print("0x%x:\t%s\t%s" %(i.address, i.mnemonic, i.op_str))


legals = 'abcdefghijklmnopqrstuvwxyz \t\n#ABCDEFGHIJKLMNOPQRSTUVWXYZ_0123456789'
for i in legals:
    f(i)
```

I got the following results:
- 


## Summary
This year's Binary Golf is fun, albeit a bit too "cheaty" in my opinion - it's easy to "game the system" with external dependencies.  
However, perhaps that's also a nice goal - and that's definitely something that I'd call "a hacker's mindset".  
I might try to have more fun if I have time (`bootloader` or `COM files` are on my mind), but I think this was a nice opportunity to show Linux shellcoding approaches and also discuss TSX a bit.

Stay tuned!

Jonathan Bar Or
