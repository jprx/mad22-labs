# Challenges for MAD 22

## ASLR Bypasses

=================================================
### Egghunter 100

Break ASLR with an egghunter!

Egghunters are a technique used to sidestep ASLR using info leaked by a syscall. Certain Linux syscalls will return a different result via `errno` (see: `man errno`) depending on whether the address is mapped or not.

For this exercise, use the `access` syscall.

`access` will return `EFAULT` if the first argument is unmapped.

Call `access` on each address in the range until you find one that is mapped (that is, `access` doesn't return `EFAULT`)

### Challenge Files

`aslr-chals/src/egghunter.c`

### To test your code
In `aslr-chals/src`

`make`

`./egghunter`

### Incorrect output
```
Incorrect :(
It was: 0x13C20EEF000
You said: 0x0
```

### Correct output
```
Correct!!!
It was: 0x5F53651F000
You said: 0x5F53651F000
```

### Access the server

`ssh access-code@unicorn.csail.mit.edu`

=================================================
### Prefetch 350

Same deal as Egghunter, except now `access()` is blocked! (Try your Egghunter code here, you should see `Bad system call` printed).

Let's find a way to bypass ASLR without making any system calls!

The [`prefetch`](https://c9x.me/x86/html/file_module_x86_id_252.html) instruction provides a hint to load a particular line into the cache. We will use the `prefetch` instruction to try and load every address into the cache, observing how long each one takes to load. This technique was proposed in *Prefetch Side-Channel Attacks: Bypassing SMAP and Kernel ASLR* [1] (Specifically Section 3.2's "Translation-Level Oracle").

The `prefetch` instruction will try to translate the given virtual address into a physical address and load it into the cache hierarchy. If the address is unmapped, it will require a full page table walk (which takes many cycles!). If the page is already present in the cache hierarchy, `prefetch` will stop early.

Timing the `prefetch` instruction is a little tricky due to CPU synchronization. We recommend you follow the approach used by the paper authors:

```bash
mfence
rdtscp
cpuid
prefetch
cpuid
rdtscp
mfence
```

While doing this exercise, you may find referring to the [source code for the prefetch paper](https://github.com/IAIK/prefetch/blob/master/addrspace/addrspace.c) helpful [2].

### Challenge Files

`aslr-chals/src/prefetch.c`

### To test your code
In `aslr-chals/src`

`make`

`./prefetch`

### Incorrect output
```
Incorrect :(
It was: 0x13C20EEF000
You said: 0x0
```

### Correct output
```
Correct!!!
It was: 0x5F53651F000
You said: 0x5F53651F000
```

### References
[1] Daniel Gruss et al. Prefetch Side-Channel Attacks: Bypassing SMAP and Kernel ASLR. 2016. DOI:https://doi.org/10.1145/2976749.2978356

[2] [IAIK Prefetch Paper Code](https://github.com/IAIK/prefetch/blob/master/addrspace/addrspace.c)

## Transient Execution

=================================================
### Flushed Away 100

Use `flush+reload` to leak a secret from kernel memory across privilege levels!

Recall `flush+reload` takes the following 3 steps:

1. Attacker flushes a cache line with `clflush`
1. The victim may (or may not) load the line
1. The attacker watches how long it takes to reload the line

If the line loads fast, the victim accessed it. Otherwise, the victim didn't! We found that **175 cycles** is the best number to use for distinguishing load times on the `arch-sec` machines (however, feel free to adjust this number if you'd like to).

We have provided several helper methods in `inc/lab2.h`:

`clflush(addr)`: Flush `addr` from the cache system using the `clflush` instruction.
`time_access(addr)`: Reports how many cycles it takes to load `addr`.

This is the victim code that runs in the kernel:

```
def victim_part1(shared_mem, offset):
    secret_data = part1_secret[offset]
    load shared_mem[4096 * secret_data]
```

Note that loads are scaled by a page (4096 bytes). So if the secret is 0, page 0 will be loaded. If the secret is 1, then page 1 will be loaded. And so on.

Since the attacker and victim share memory, we can use `flush+reload` (instead of other techniques such as `prime+probe`).

### Challenge Files

`spectre-chals/part1-src/attacker-part1.c`

### To test your code
In `spectre-chals/`

`make`

`./part1`

### Correct Output
On correct execution your code should print the flag leaked from kernel memory. It will look something like this:

`mad{...}`

=================================================
### Basic Spectre 250

Now that we have `flush+reload` working, let's mix in Spectre!

Here's the new victim:

```
# Leak the first part of the flag ("mad{") non-speculatively:
part2_limit = 4

def victim_part2 (shared_mem, offset):
  secret_data = part2_secret[offset]
  mem_index = 4096 * secret_data

  # Make the if statement take longer
  # (This makes your life easier)
  flush(part2_limit)

  if offset < part2_limit:
    load shared_mem[mem_index]
````

This is very similar to "Flushed Away" (in fact, your existing code should still work for the first 4 characters without modifications).

However, now leaking the 5th character and on will require using Spectre to bypass the bounds check.

Use the following outline to guide your attack:

1. Train the branch predictor by calling the victim with values to make the if statment true.
1. Flush the shared memory.
1. Run the victim out of bounds.
1. Reload shared memory to learn what was accessed.

The only additional step here is the training step.

### Challenge Files

`spectre-chals/part2-src/attacker-part2.c`

### To test your code
In `spectre-chals/`

`make`

`./part2`

### Correct Output
On correct execution your code should print the flag leaked from kernel memory. It will look something like this:

`mad{...}`

=================================================
### Advanced Spectre 500

The training wheels have officially come off! This is a realistic (and difficult) speculative execution attack!

Your code from part 2 should no longer work here (try it!).

Here's the final victim:

```
part3_limit = 4
def victim_part3 (shared_mem, offset):
  if offset < part3_limit:
    false_dependency = long latency computation resulting in 0 secret_data = part3_secret[offset]
    mem_index = 4096 * secret_data
    load shared_mem[mem_index + false_dependency]
````

By adding a long latency dependence on the memory access and removing the convenient flush from the victim code,
we've made the victim code harder to attack (as now the load happens significantly later).

Is there something we can do to make the branch take longer to resolve?

### Challenge Files

`spectre-chals/part3-src/attacker-part3.c`

### To test your code
In `spectre-chals/`

`make`

`./part3`

### Correct Output
On correct execution your code should print the flag leaked from kernel memory. It will look something like this:

`mad{...}`

# Hint

If the entire cache was emptied, the victim would need to reload part3_limit, which would be slow.

# Hint 2

You can empty the whole cache by doing lots of loads!

