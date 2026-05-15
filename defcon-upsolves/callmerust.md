# Callmerust
This challenge consists of a loader written in C and a library written in Rust. You provide rust code, and then the loader runs & ptraces it.

## The loader
The loader does a few interesting things with your rust code. First, it blocks all memory-unsafe features. It also includes a file called `lib.rs`. 
After compiling and running the rust code, it will ptrace the child process and check for a certain set of bytes at the current instruction point after an `int3` instruction is hit.
```c
                        if(data!=0xcc0000000000841f){
                            kill_all_and_exit(fd, cpid, status);
                        }
                        data = ptrace(PTRACE_PEEKDATA, child, (void*)(regs.rip-8-1), NULL);
#ifdef DEBUGF
                        fprintf(fd, "--------> %lx\n", data); fflush(fd);
#endif
                        if(data!=0x0000000000841f0f){
                            kill_all_and_exit(fd, cpid, status);
                        }

```
After this is detected and the check is passed, it will replace any strings in the child process's memory matching "FLAG" with the flag. The main issue now becomes the file `lib.rs`. 
Inside its only public function, `zzz`, it contains an inline assembly block with the right instructions to pass the check, inside a panic handler. This checks against two globals which need to be set by the function.
```rust
                if unsafe{V && S}{
                    if *cc == unsafe{M}{
                        println!("!");
                        unsafe{
                            asm!(".byte 0x0F, 0x1F, 0x84, 0x00, 0x00, 0x00, 0x00, 0x00; int3;");
                            for &(f,s) in data.iter() { println!("{:#x}", *((((rr(f) as u64)<<16)+((s&0x000fffff) as u64)) as *const u128)); }
                            asm!("mov rax, 0x0; jmp rax");
                            println!("{}", *(FLAG.as_ptr()) as char);
                        }
                    }
                }

```
After a couple hours of slopping and staring at bullshit obfuscation, we can pass the checks and set the two globals. There's a third condition inside zzz to trigger the panic handler, but we can just bypass that with `std::panic`. 
The next big obstacle is the `rr` function. Long story short, it's some crypto BS that I didn't understand, so what we do is instead of using the memory read that has to go through `rr`, we just abuse `/proc/self/mem`. 
## Final solve
- Pass checks in `zzz` to set the V and S globals
- Scan through `/proc/self/mem` and replace the `asm!("mov rax, 0x0; jmp rax");` assembly with `nop`s so we can return from `zzz` after getting the flag inserted into our process.
- Scan through `/proc/self/mem` for the flag

Now that I think about it, I didn't need to do **anything** involving the library, I could have just abused `/proc/self/mem` for the whole thing. Oh well. 
- Scan through `/proc/self/mem` and replace 

