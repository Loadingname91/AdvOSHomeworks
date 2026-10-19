# HW1 answers

Fill in this file and hand it in with your code. Replace each
`(your answer)` with two or three sentences. Keep the question numbers
(part.question), and do not delete the questions.

- **Name:** Hitesh Balegar - u1522135 
- **Machine and environment you tested on** (CADE lab machine with QEMU, or Bochs): CADE lab1-5.eng.utah.edu, QEMU 9.0.4 with KVM (`make qemu-nox`, 2 and 4 CPUs)
- **How far did you get?** (for example "all six parts work", or "stuck in part 4, block B"): All six parts work. `runvm helloguest` prints the greeting and halts at 0x10004E on repeated runs, including with CPUS=4. Check output in `output.png`

## Part 1: Entering VMX operation

### Question 1.1

> In Step 1, why does the function `return` when VMX is missing instead
> of calling `panic()`?

**Answer:** A CPU without VMX isn't a kernel bug. It only means this CPU can't run guests. If I called panic(), xv6 wouldn't boot at all on such a machine (for example QEMU without KVM), even though everything else in the OS works. Returning leaves `vmx_ok` clear, so later `vmcreate()` refuses cleanly and runvm prints `vmcreate failed`.

### Question 1.2

> In Step 2, on most machines the firmware has already enabled and
> locked VMX. Why does the code still handle the unlocked case instead
> of just failing?

**Answer:** Some environments, such as emulators, nested hypervisors or firmware that skips this setup, leave IA32_FEATURE_CONTROL unlocked. While it's unlocked VMXON faults, but software is still allowed to write the MSR, so I set the lock bit and the "VMX outside SMX" bit myself. Failing there would turn VMX off on a machine that can actually use it.

### Question 1.3

> In Step 3, what would happen if we skipped `lcr4(cr4)` and went
> straight to `VMXON`?

**Answer:** CR4.VMXE would still be 0 (and CR0/CR4 might break the fixed-bit rules), and VMXON with CR4.VMXE = 0 raises an invalid-opcode exception (#UD) instead of setting the failure flags. The `vmxon()` wrapper only checks flags, so the fault would go to xv6's trap handler and panic the kernel at boot rather than printing `vmxon failed`.

### Question 1.4

> In Step 4, why does each CPU need its own VMXON region? What could go
> wrong if all CPUs shared one?

**Answer:** The VMXON region is memory the CPU uses for its own VMX bookkeeping while it's in VMX operation, and the SDM says each logical processor should use a separate one. If all CPUs shared one page they could overwrite each other's internal state, which the manual leaves undefined, so the failures would be random and hard to trace. Each CPU enters VMX on its own, so each gets its own `kalloc()` page.

## Part 2: Creating a VM, its memory, and the EPT

### Question 2.1

> An ordinary x86-64 page table translates virtual addresses to physical
> addresses. What does the EPT translate, and who controls it? What
> would go wrong if the hypervisor gave the guest direct access to the
> host's physical memory instead?

**Answer:** The EPT translates guest-physical addresses (what the guest thinks is physical memory) into host-physical addresses, and only the hypervisor controls it. The guest's own page tables only do the first step, guest-virtual to guest-physical. If the guest could use host physical memory directly it could read or overwrite the xv6 kernel, other processes and other VMs; with EPT, touching anything I didn't map causes an EPT violation exit instead.

### Question 2.2

> In Step B7 we zero every page of guest memory, and in `eptwalk()` we
> zero every new table. These are two different reasons to zero. State
> each, and say what the guest or the CPU would see if we skipped it.

**Answer:** Guest pages are zeroed so the guest can't read old host data (kfree fills pages with 0x01 junk, and a reused page may hold kernel or another process's data) and so its uninitialized data starts at zero, since the flat binary has no loader to clear it. New EPT tables are zeroed because a zero entry means "not present": a junk entry with bit 0 set would look present, so the CPU would translate through a random address (wrong memory or an EPT misconfiguration exit) and `eptwalk()`/`eptfree()` would follow it and could free pages they never allocated.

### Question 2.3

> In Step B8, why does `vmcreate()` copy the image page by page through
> `eptwalk()`, instead of one `memmove` of `len` bytes to the guest's
> first page?

**Answer:** The guest-physical range at 1 MB is contiguous, but every 4 KB guest page is backed by its own `kalloc()` page, which can be anywhere in host memory and in any order. One `memmove` of `len` bytes would run off the end of the first host page into unrelated kernel memory. So I translate each guest page through the EPT and copy at most one page at a time.

### Question 2.4

> An EPT entry is "present" if any of R, W or X is set, not if one
> particular bit is set. A common bug is to test only bit 0. Name one
> kind of entry that this bug would treat as missing but that the CPU
> treats as valid (hint: see Appendix A.10 for execute-only support), and
> say why our code never creates such an entry.

**Answer:** An execute-only entry (X = 1, R = 0, W = 0) has bit 0 clear, so a bit-0 test calls it missing, while a CPU that supports execute-only translations (Appendix A.10, bit 0) treats it as a valid mapping. Our code never creates one because every entry it writes, upper level or leaf, has all three bits set (`EPT_RWX`).

## Part 3: The VMCS, describing the guest and the host

### Question 3.1

> In `adjustctl()`, why is it enough to compute
> `(want | low) & high`? What does the function do in the case where
> `want` has a bit that `high` has cleared, and why is returning an error
> better than silently dropping the bit?

**Answer:** The low half lists bits the CPU requires to be 1, so ORing it in adds them, and the high half lists bits the CPU allows to be 1, so ANDing with it removes anything unsupported; what's left is a legal control word. If `want` has a bit that `high` clears, `(v & want) != want`, so `adjustctl()` prints the missing bits and returns -1 and `vmcreate()` fails. Dropping the bit silently would build a VM without something we rely on (without HLT exiting the guest would really halt the CPU, for example), and I'd only find out later from a confusing failure.

### Question 3.2

> The exception bitmap is all ones. If it were 0, what would happen to a
> page fault in the guest, step by step, given that the guest's IDTR limit
> is 0? Why is the all-ones setting more useful to someone debugging a
> guest?

**Answer:** With a zero bitmap the page fault is delivered through the guest's IDT. The IDTR limit is 0, so vector 14 is outside the table and delivering it raises #GP; delivering the #GP fails the same way and becomes a double fault, and when that fails too it becomes a triple fault, which causes a VM exit (reason 2). runvm then only says `guest triple fault`, with no vector or error code. With all ones, the very first exception exits with its vector, error code and RIP, so I can see what actually went wrong.

### Question 3.3

> Why must "enable EPT" be 1 whenever "unrestricted guest" is 1? Think
> about what the CPU does with a guest linear address when `CR0.PG = 0`
> (SDM 28.6).

**Answer:** With CR0.PG = 0 the guest's linear address is used directly as a guest-physical address. Without EPT nothing would translate it further, so it would be used as a host-physical address and the guest would be touching real host memory with no isolation. EPT is what maps guest-physical to host-physical, which is why VM entry requires "enable EPT" whenever "unrestricted guest" is set.

### Question 3.4

> The real `CR4` in the VMCS has `VMXE` = 1 but the guest reads it as 0.
> Which two VMCS fields make that work? What happens if the guest tries to
> set `CR4.VMXE` to 1, and why do we not let it?

**Answer:** `CR4_GUEST_HOST_MASK`, with the VMXE bit set so the host owns that bit, and `CR4_READ_SHADOW`, with VMXE = 0, which is what the guest's reads of CR4 return. If the guest writes CR4 with VMXE = 1, the value differs from the shadow in a host-owned bit, so the CPU exits (control-register access, reason 28) instead of doing the write; our kernel doesn't handle that, so runvm reports an unhandled exit. We don't support nested VMX, and the real CR4.VMXE has to stay 1 while the CPU is in VMX operation anyway, so the guest's view is kept separate from the real bit.

## Part 4: Entering and leaving the guest

### Question 4.1

> `struct guestregs` has no `rsp` field. Why can the guest's stack
> pointer not be handled the way the other registers are? Where does the
> CPU keep it instead, and what would go wrong if `vmx_exit` tried to
> store `%rsp` into `*regs` like the others?

**Answer:** The CPU saves the guest's RSP into the VMCS (`GUEST_RSP`) on every exit and loads it from there on entry, together with RIP and RFLAGS; vmrun copies it to and from `vm->rsp`. By the time vmx_exit runs, the CPU has already replaced %rsp with `HOST_RSP`, so the guest's stack pointer isn't in the register any more. Storing %rsp into `*regs` would save the host's kernel stack address as the guest's.

### Question 4.2

> `HOST_RIP` is written once, in `vmcsinit()`, but `HOST_RSP` is written
> in `vmx_enter` on every entry. What makes one constant and the other
> not? Describe a concrete situation in which the `HOST_RSP` written by
> a previous entry would be wrong.

**Answer:** `HOST_RIP` is the address of the vmx_exit label, fixed when the kernel is linked. `HOST_RSP` is wherever vmx_enter's frame sits on the current kernel stack, which depends on whose kernel stack it is and how deep the call chain is. If vmx_enter were reached at a different depth than last time (vmrun's frame changing size, or a call through another function), an old `HOST_RSP` would point into a stale frame, and vmx_exit would pop garbage into the callee-saved registers and `ret` to a garbage address, crashing the kernel. Writing it on every entry costs one VMWRITE and removes that assumption.

### Question 4.3

> `vmrun()` always calls `vmx_enter(&vm->regs, 0)`, and so always uses
> `VMLAUNCH`. Which instruction clears the launch state, and where does
> `vmrun()` execute it? What would the VM-instruction error be if the
> code passed `launched = 1` on the first entry?

**Answer:** VMCLEAR sets the launch state back to "clear". vmrun executes it right after reading the exit information and before `popcli()` (and vmcreate does it after `vmcsinit()`), so every entry sees a clear VMCS. Passing `launched = 1` on the first entry would execute VMRESUME on a clear VMCS, which fails with VM-instruction error 5, "VMRESUME with non-launched VMCS".

### Question 4.4

> `vmxhoststate()` is called before every entry, not once. Name two
> fields it writes that can differ between two calls to `vmrun()`, and
> say what is different (CPU, process, or both) for each. What would
> happen at the next VM exit if the field kept the old value?

**Answer:** `HOST_CR3` depends on the process: it's the page table of the process calling vmrun. `HOST_TR_BASE` (and `HOST_GDTR_BASE`) depend on the CPU: they point at this CPU's TSS and GDT, and after VMCLEAR and `popcli()` the process can resume on a different CPU. With a stale CR3 the exit would land in another process's page table, possibly one already freed; with a stale TR base this CPU would use another CPU's TSS, so the next interrupt would switch to the other CPU's kernel stack and two CPUs could end up on the same stack.

## Part 5: Running the guest, the vmrun loop and exit handling

### Question 5.1

> In Step 8, `vmclear()` must come before `popcli()`. Describe a
> concrete sequence of events, with two CPUs, in which the opposite
> order corrupts the VMCS. Which sentence of SDM 27.11.1 do you rely on?

**Answer:** CPU 0 does VMPTRLD, runs the guest and takes an exit, so the VMCS is active on CPU 0 and parts of it may be cached inside that processor. With `popcli()` first, a timer interrupt arrives and yields, the process is scheduled on CPU 1, and CPU 1 runs the VMCLEAR and on the next pass VMPTRLD and VMLAUNCH while the VMCS is still active on CPU 0. Later CPU 0 flushes its cached copy back to memory and overwrites the state CPU 1 wrote. I rely on SDM 27.11.1: "If a VMCS is to be “migrated” from one logical processor to another, the first logical processor should execute VMCLEAR for the VMCS (to make it inactive on that logical processor and to ensure that all VMCS data are in memory) before the other logical processor executes VMPTRLD for the VMCS (to make it active on the second logical processor)." The same paragraph warns that "A VMCS that is made active on more than one logical processor may become corrupted."

### Question 5.2

> We pass `launched = 0` to `vmx_enter` on every entry, so we always use
> `VMLAUNCH`. What would the program have to do differently if we kept
> the VMCS loaded between exits and used `VMRESUME` instead? What would
> we give up?

**Answer:** The VMCS would stay current on a CPU between vmrun calls, so I'd have to catch migrations: VMCLEAR it on the old CPU when the scheduler switches the process out (a hook in the scheduler, like KVM's preempt notifiers), VMPTRLD it on the new CPU, and track whether it's launched so I pick VMLAUNCH after a clear and VMRESUME otherwise. Host state would then only need rewriting after a migration. I'd gain faster round trips but give up the simple invariant that no VMCS is cached anywhere outside vmrun, which keeps freeing a VM and reading its registers easy.

### Question 5.3

> On an external-interrupt exit, `vmxhandle()` does nothing but return 1,
> yet the interrupt is handled by the host before the guest runs again.
> Where and when exactly does the host handle it? What would change if
> the exit control "acknowledge interrupt on exit" were set?

**Answer:** Because "acknowledge interrupt on exit" is off, the interrupt stays pending in the local APIC, and the exit leaves interrupts disabled. vmrun keeps them off until the `popcli()` right after VMCLEAR; at that moment the CPU takes the pending interrupt through xv6's normal IDT and trap handler (the timer may even reschedule us), and only after that does `vmxhandle()` return 1 and the loop re-enter. With acknowledge-on-exit the CPU would accept the interrupt during the exit and put its vector in the exit interruption-information field, so it would no longer be pending and the kernel would have to dispatch it itself, or the interrupt would be lost.

### Question 5.4

> The `WRMSR` exit and the `OUT` exit are both followed by
> `vm->rip += inslen`. What would happen if the code used a fixed
> length (for instance 2) instead? Name one instruction in `helloguestasm.S`
> or `helloguest.c` that shows the problem.

**Answer:** Instructions have different lengths, so a fixed 2 only works for some. WRMSR and RDMSR in helloguestasm.S are 2 bytes and would happen to work, but the `outb` in helloguest.c assembles to `outb %al, (%dx)` (port 0x3F8 doesn't fit in an immediate, so it goes through DX), which is a single byte. Skipping 2 bytes would resume the guest in the middle of the next instruction, and it would run garbage and fault or misbehave.

## Part 6: The other half: the monitor (runvm) and the guest

### Question 6.1

> In `doio()`, why does an `IN` need the monitor to write the guest's
> RAX with `vmsetreg()`, instead of `doio()` just returning a value?
> And why does `completein()` read RAX with `vmgetreg()` for a 1-byte
> `IN` but not for a 4-byte one?

**Answer:** The IN's result has to end up in the guest's RAX, and the guest's registers live in the kernel's `struct vm`, not in runvm; a value returned by `doio()` would stay in user space and never reach the guest. `vmsetreg()` is the one way runvm changes guest registers, so the kernel doesn't have to keep any pending-IN state. A 4-byte IN writes EAX, which zero-extends into all of RAX, so the old value doesn't matter, while a 1-byte IN only replaces AL and keeps the other seven bytes, so `completein()` has to read RAX first and merge.

### Question 6.2

> `helloguest.ld` links the guest at `0x100000`. Suppose you change it
> to `0x200000` but leave `VM_LOAD_ADDR` alone. Name two things in the
> guest that will go wrong and say why. (Look at the page tables, the
> `gdtdesc`, and where the first instruction is.)

**Answer:** runvm still starts the guest at `VM_LOAD_ADDR` (0x100000), where the image is copied, so the first instructions run, but every absolute address in the guest now points 1 MB too high, into zeroed memory. `movl $pml4, %eax` loads CR3 with the wrong address and the page-table entries themselves (`.quad pdpt + 0x3`) hold 0x2xxxxx addresses, so once CR0.PG is set the CPU finds empty tables and the next fetch page-faults (`guest exception 14`). `lgdt gdtdesc` also reads its descriptor from the wrong place (zeros), and `ljmp $8, $start64` and `call guestmain` jump to 0x2xxxxx, where there's no code.

### Question 6.3

> The guest in Step 8 sets `CR4.PAE` before `CR0.PG`. What happens if
> you swap the order of these steps? Then, for the Checkpoint's
> `runvm helloguest 1` experiment: which message do you get, and which
> check in the kernel produces it?

**Answer:** The guest sets EFER.LME before paging, so turning on CR0.PG while LME = 1 and CR4.PAE = 0 is illegal and raises #GP; with the exception bitmap all ones, runvm prints `guest exception 13`. For `runvm helloguest 1` I get `runvm: vmcreate failed`. It comes from the first check in `vmcreate()` (Step B1), `VM_LOAD_ADDR + len > memsz`: 1 MB of memory ends exactly where the image would start.

### Question 6.4

> Each `OUT` in the guest causes a VM exit, a trip through the kernel,
> and a return to `runvm`, which calls `write()`. Why did the design
> put the serial port in `runvm` and not in the kernel? What would you
> gain, and what would you lose, by emulating it inside the kernel?

**Answer:** Keeping the serial port in runvm keeps the kernel small, like KVM: the kernel runs the guest and handles only what it must, a bug in device code crashes runvm instead of xv6, and I can change devices without rebuilding the kernel. Emulating it in the kernel would save a trip to user space and a `write()` per character, 24 round trips for this greeting, which adds up for a chatty device. In exchange the kernel would get more code that the guest can poke at, and less isolation and flexibility.
