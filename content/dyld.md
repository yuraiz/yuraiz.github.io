+++
title = "How thread locals work on macOS"
date = 2026-05-30
+++

I've spent significant time figuring out how to implement thread local inspection support for a debugger on macOS so I hope this article may be helpful for someone.

If you want to understand thread locals from the point of a static linker there's a good [article by Jakub Konka](https://www.jakubkonka.com/2021/01/21/llvm-tls-apple-silicon.html). I'm not currently interested in the Intel macs, so the article touches only Apple Silicon.

Let's start with a simple code example:

```c
#include <stdio.h>

thread_local int first = 42;
thread_local int second = 390;

int main() {
    int* thread_local_addr1 = &first;
    int* thread_local_addr2 = &second;

    printf("The addr of the 1st thread local: %p\n", thread_local_addr1);
    printf("                2nd thread local: %p\n", thread_local_addr2);

    return 0;
}
```

You can compile it like

```bash
$ clang -std=c23 thread_locals.c -g -o thread_locals
```

(I use C23 to get the `thread_local` keyword, otherwise you may just use `__thread` on Clang)

Let's step into the code using a debugger:

![disassembly of the computation of the thread local address](/dyld/step-into.png)

That code stores the `__thread_vars` section address to `x0`, adds an offset that's a multiple of 24 and calls the function by the address from in first 8 bytes of the value.

We can read the memory at the address:

![memory view](/dyld/memory.png)

You may notice that the value `88 83 08 8e 01` is repeated after 24 bytes from the start. That's the address of the function we branch to.

If we disassemble that address we get the following code:

![disassembly of the code by the memory address](/dyld/dyld.png)

That's actually the code of `__tlv_get_addr` from dyld, and the source code is [available on github][dyld-asm]. Note that all of the code snippets from dyld and xnu are simplified to include only the ARM64 parts of the code:

```
// Parameters: x0 = descriptor
// Result:  x0 = address of TLV
__tlv_get_addr:
ldr  w16, [x0, #8]           // get key from descriptor (TLV_Thunkv2.key)
mrs  x17, TPIDRRO_EL0
and  x17, x17, #-8           // clear low 3 bits
ldr  x17, [x17, x16, lsl #3] // get thread allocation address for this key
cbz  x17, LlazyAllocate      // if NULL, lazily allocate
ldr  w16, [x0, #12]          // get offset from descriptor (TLV_Thunkv2.offset)
add  x0, x17, x16            // return allocation+offset
ret  lr
```

x0 points to a structure like [that][dyld-thunk]:

```c
// runtime structure of 64-bit arch thread-local thunk
struct TLV_Thunkv2
{
    // points to __tlv_get_addr
    void*    func;
    // pthread key
    uint32_t key;
    // offset from the start of the memory allocation
    uint32_t offset;
    // if zero, then content is all zeros,
    // otherwise offset from the address of the field
    int32_t  initialContentDelta;
    // the size of the allocation block
    uint32_t initialContentSize;
};
```

The only thing LlazyAllocate does is store the registers and call `ThreadLocalVariables::instantiateVariable` that allocates and initializes the memory block:

```c
void* ThreadLocalVariables::instantiateVariable(const Thunk& thunk)
{
    void*             buffer = nullptr;
    dyld_thread_key_t key    = 0;

    TLV_Thunkv2* thunkv2 = (TLV_Thunkv2*)&thunk;
    key = thunkv2->key;
    if (thunkv2->initialContentDelta != 0) {
        // initial content of thread-locals is non-zero so copy initial bytes from template
        buffer = malloc(thunkv2->initialContentSize);
        const uint8_t* initialContent = (uint8_t*)(&thunkv2->initialContentDelta) +
                                        thunkv2->initialContentDelta;
        memcpy(buffer, initialContent, thunkv2->initialContentSize);
    }
    else {
        // initial content of thread-locals is all zeros
        buffer = calloc(thunkv2->initialContentSize, 1);
    }

    // set this thread's value for key to be the new buffer.
    dyld_thread_setspecific(key, buffer);

    return buffer;
}
```

A single memory allocation is used for both of the thread locals here, I don't know how it is decided to split thread locals into separate sections.

Now you've seen (most of) the code the program executes to get an address for a thread local. But how does one compute the address in a debugger?

Well, the debug info actually provides the address of TLV_Thunk2 relative to the start of the module:

![dwarfdump shows thread local location](/dyld/dwarf.png)

So the only thing you need to do in a debugger is to read the pointer to `__tlv_get_addr` and to evaluate it, like LLDB does.

I didn't want to implement function calling just to support thread locals, so I figured out an alternative implementation:

The hardest thing was getting the value of the tpidrro_el0 system register. The problem is that there is no API to read system register state for a thread, but it so happens that thread_identifier_info->thread_handle contains the same address. It points to the pthread's internal "thread specific data slots" array. I found this out after reading the code of the kernel and pthreads.

You can get the value like that:

```c
struct thread_identifier_info identifier_info;
mach_msg_type_number_t count = THREAD_IDENTIFIER_INFO_COUNT;
kern_return_t kr = thread_info(thread_port,
                               THREAD_IDENTIFIER_INFO,
                               (thread_info_t)&identifier_info,
                               &count);

uint64_t tsd_vaddr = identifier_info.thread_handle;
```

The rest is comparatively trivial:

```c
uint64_t thunk_offset = // ...
uint64_t tsd_vaddr    = // ...
uint64_t result       = 0;

typedef struct
{
    uint64_t func;
    uint32_t key;
    uint32_t offset;
    int32_t init_delta;
    uint32_t init_size;
} TLV_Thunkv2;

TLV_Thunkv2 tlv_thunk = {0};
// assuming the memory reading is already implemented
task_mem_read(task, &tlv_thunk, thunk_off, sizeof(tlv_thunk));

uint64_t base_vaddr = tsd_vaddr & 0xfffffffffffffff8;

uint64_t allocation_vaddr = 0;
uint64_t alloc_off  = base_vaddr + (tlv_thunk.key << 3);
task_mem_read(task, &allocation_vaddr, alloc_off, sizeof(uint64_t));

if(allocation_vaddr != 0) {
    // found the allocation address
    result = allocation_vaddr + tlv_thunk.offset;
} else if(tlv_thunk.init_delta != 0) {
    // not allocated yet, use the initial value
    result = thunk_off +
        offsetof(TLV_Thunkv2, init_delta) +
        tlv_thunk.init_delta +
        tlv_thunk.offset;
} else {
    // not allocated yet, the initial value is 0
    result = 0;
}

return result;
```

I used that approach for my WIP port of RADDBG and it works.

![RADDBG shows the thread local value](/dyld/debug.png)

Hope that was interesting!

[dyld-asm]: https://github.com/apple-oss-distributions/dyld/blob/e9da5ae571b4191dcdbcfd827363f19847c84561/libdyld/threadLocalHelpers.s#L221
[dyld-thunk]: https://github.com/apple-oss-distributions/dyld/blob/e9da5ae571b4191dcdbcfd827363f19847c84561/libdyld/ThreadLocalVariables.h#L146
