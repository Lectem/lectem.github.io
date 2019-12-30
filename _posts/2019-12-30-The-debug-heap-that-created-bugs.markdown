---
layout: post
title:  "The Debug heap that created bugs"
categories: windows heap appverifier detours
hidden: true
---

Aka:Story of one man-month of development lost due to a bug in a debugging tool.

![ackiechan-meme](jackiechan-meme.jpg)

Memory corruption can be hard to track, especially when you have a multithreaded application with features that makes it hard to reproduce.
This is generally the case for games, where thousands of factors can make a race condition appear or not. Framerate could be unstable, the GPU could take a bit more time than usual to render, something being streamed for the internet took a bit longer than usual.
Even without all those reasons, you might want tools that help you debug those issues. 
There are quite a few of them available that can help you detect memory corruption and race condtions.

One such tool on Windows is [AppVerifier](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/application-verifier), it has a good number of features and one of those is the debug heap. 
It seems that it does a bit more than just enabling the gflags [Page Heap Verification](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/enable-page-heap), and can hook functions such as HeapAlloc.

The Page Heap can be used not only to track memory corruption but also leaks, and for any application using new/malloc/HeapAlloc for allocations. More info on how to use it from windbg [here](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-heap). 
Sounds great right? I have been using this tool for a while now. But the last time I used it, I had issues finding where the bug it detected came from.

I usually fire up WinDbg to find the cause of the crash, so that I can easily inspect the memory that was corrupted.
Most of the time, the crash happened in DirectX or in the GPU drivers. Usually, this does not sound good. I also sometimes had this particular crash in libcurl.
For a while I couldn't figure out what the issue was, callstacks had nothing in common but somehow, it was not random.
A few callstacks kept appearing in those two libraries. At first, I thought we might have had some incompatibility in libcurl with it's allocations. I made sure we compiled it with the correct flag, set the allocation functions at init. And bam, no more issues.

It seemed like a weird fix since curl allocates everything using its own functions, and is wrapped in its own .dll. 
But the bug disappeared. And as our QA didn't reproduce it unless they used AppVerifier we decided to stop worrying about it. I spent about a week or two looking for various causes for the crash, found potential bugs in other parts so it was kind of worth it anyway.

Until we had another memory corruption. As usual I fired up AppVerifier and WinDbg. After running the game a few times, I found the crash we were looking for. However I did encounter once the old DirectX crash. It was not gone.
So I fixed one issue. Then another. Used static analysis tools, other applications such as Intel Inspector, coded my own page guards, but nothing did it. I kept disabling gameplay features, but the bug stayed. And was most of the time getting harder to reproduce. 

I did have a few suspects though, I knew that the problem was that memory allocated through `HeapAlloc` with `HEAP_ZERO_MEMORY` had parts not set to zero. And the issue I had in libcurl was similar: memory allocated with `calloc` was not zero-filled.
I looked for ages where this could be triggered from. Most of the time, the value was filled with `0xCC`, or `0xD0`. Which respecitvely meant uninitialized memory or freed memory.

And so I assumed there was a use after free somewhere.
My hypothesis was the following:

- Thread A frees the memory
- Thread B allocates memory, asking it to be zero-filled, and gets the same address
- Thread A does a use-after-free
- Thread B reads memory that should have been null but is not, and we get an invalid pointer.

And that's where I made a mistake. I thought that my tooling was bug-free. But let's get back to the problem.

At some point, unable to find the source of the bug, I sent a mail to the DirectX team with a (huge) crashdump. At first, they told me it probably was a mistake on our side. That's decent right, DirectX is used in a lot of games and never showed such issue. And it only happened when using debug tools such as AppVerifier. 
But as I explained the problem more deeply, my contact understood the issue and was kind enough to investigate a bit. Sadly for me, he confirmed that the problem was not in DirectX. However, what we noticed was that the memory was always allocated by [HeapAlloc](https://docs.microsoft.com/en-us/windows/win32/api/heapapi/nf-heapapi-heapalloc) with the `HEAP_ZERO_MEMORY` which confirmed my earlier suspicions.

Something I did not notice however, was that the keyword "airport" was in the memory block. Most of the time. I dismissed it at first as I already the checked the only part in our code that could write such string. Or should I say array of strings. And we had a lot of those (which made me realize we really needed a better way to store those, as we got in the 40k duplicated strings in some cases!), that were stored in a dynamic array that was getting reallocated.
And so I started to play with it, changing one letter for another before and after reallocation.

That's how we came to the conclusion that something was wrong in the windows heap. We couldn't believe it, yet here it was.
The weird part was that it didn't seem to happen without the debug heap enabled. Or was it?
So we sent a mail to the AppVerifier owner, which didn't really believe us at the beggining. This is fair since nobody ever reported this for ages. He tried a small sample and could not reproduce. A single-threaded sample, yet our issue was a multi-threading issue.

During the same period, I was learning how to use Microsoft's [Detours](https://github.com/Microsoft/Detours) library for a side project of my own. I then decided to put this to profit, and detour the HeapAlloc function, adding a simple yet effective:

```cpp

DECLSPEC_ALLOCATOR LPVOID MyHeapAlloc(
  HANDLE hHeap,
  DWORD  dwFlags,
  SIZE_T dwBytes
)
{
    LPVOID memoryBlock = RealHeapAlloc(hHeap,dwFlags,dwBytes);
    const char* pMem = (const char*)memoryBlock;
    if(dwFlags & HEAP_ZERO_MEMORY)
    {
        for (size_t i = 0; i < cb; ++i)
        {
            if (pMem[i] != 0)
            {
                __debugbreak();
            }
        }
    }
    return memoryBlock;
}
```

Start the game with Detours, et voilà! The debug breakpoint was triggered! But only if the AppVerifier debug heap was enabled. This was enough proof for me to conclude the culprit was indeed AppVerifier, but it would be nice if we could reproduce it easily right?

So I made a minimal, reproducible example. The latest version is the following:

<script src="https://gist.github.com/Lectem/97f7687de4a4a763f9fd7ea0837fd750.js"></script>


What it does is spawn 2 threads, one that does a lot of allocations and reallocations, the other calling `HeapAlloc` with `HEAP_ZERO_MEMORY` in loop.
At first I used `aligned_*` allocation functions since that's what we were using in our game.

To enable appverifier, add your application to the list, and for the debug heap check Basics -> Heaps as shown below:

![appverifier-debug-heap](appverifier/appverifier-debug-heap.png)

You will note that AppVerifier will trigger the breakpoint for all define configurations except `AllocMethod_malloc` and `AllocMethod_ProcessHeapNoRealloc`. This is because `realloc` does not call `HeapReAlloc`. We can safely conclude that the issue happens only when calling `HeapReAlloc` in a multi-threaded context. Quite rare, but we were doing a lot of (re)allocations during loading, and allocating DirectX buffers on another thread so this triggered rather often on our codebase.  

That's why I wanted to tell you that even though it rarely happens, your debugging tools might fail you!
So always start to doubt your application, then your libraries, drivers, compiler, debugger, OS and lastly your hardware.
Those thing happens, remember that a debugging tool is still software and not always bug-free!

For people with access to the Microsoft bug database (I don't) this is registered as bug #22979026, I also hope it can get fixed at some point as AppVerifier is a very useful tool. For those who still want to use it and might have the same issue, know that you can write a small detours .dll that will do the zero-fill no matter what. 

Maybe a topic for a future post?


