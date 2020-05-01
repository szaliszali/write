# Is the Stack on the Heap?

When I start a `new Thread()`, a new stack is allocated. But where? Is it on the heap or not?

TL;DR no, but this is not the correct answer

## What Is The Stack?

Stack is a capacity-limited LIFO storage which stores *stack frames* (to keep it simple for now, these contain arguments, local variables and return pointers for nested method calls).

Would any of these properties prohibit the stack to be allocated on heap? Not necessarily.
- push and pop operations, capacity limit are trivial for managed code
- what's the case if we leave managed code for a `PInvoke` call?
    - can we share the existing managed stack?
        - we can pass current stack pointer to native code in the CPU registers, as usual
        - probably we'll be fine until
            - the allocated memory is a single continuous address block
            - GC does not move the allocated memory - we can solve this by pinning it for the scope of the call
            - we can
                - ~~ensure that the called method has enough stack space to not cause a stack overflow (no, we can't, of course)~~
                - reliably detect stack overflow
    - do we need a new stack for native calls?
        - we could, but based on the previous one, it seems unnecessary, also it would have a huge impact on performance

## Standards

What about the CLI [standard](http://www.ecma-international.org/publications/standards/Ecma-335.htm)? Not much. You need a stack, it's on you how you implement it.

> `System.StackOverflowException`: indicates that the hardware stack size has been
exceeded. The precise timing of this exception and the conditions under which it occurs
are implementation-specific. *[...]*  this exception has to do with the
implementation of that method state on physical hardware.

The JVM [specification](https://docs.oracle.com/javase/specs/jvms/se14/html/jvms-2.html#jvms-2.5.2) is also very liberal about the implementation of stack, just make sure you throw the correct type of exception.

## Detour: Detecting Stack Overflow in Native Code

Well, we can't put a cop to guard each stack, can we? Turns out, we can. With protected mode present in "advanced" microprocessors (since the Intel 80286, introduced in 1982) we can set pages of memory write-protected. So, if we sacrifice a page of memory at the end of our stack, it will protect us from overrunning the allocated area. When we push too much data in our stack, the processor will signal a page fault. This does not seem impossible with managed memory, either (although a bit complicated).

So, in theory, both are viable solutions. To find out the truth, let's take a look at the evidence.

## Where Is It?

The answer is hidden somewhere in the .Net framework implementation. Thanks to .Net being open source, we can [dig](https://source.dot.net/#q=System.Threading.Thread) into it.

We can find the `Thread` class in 3 parts.
- [Thread.Unix.cs](https://source.dot.net/#System.Private.CoreLib/Thread.Unix.cs,3980e012bae82e96) is obviously not important for us right now
- [Thread.CoreCLR.cs](https://source.dot.net/#System.Private.CoreLib/src/System/Threading/Thread.CoreCLR.cs,3980e012bae82e96) looks scary!
- [Thread.cs](https://source.dot.net/#System.Private.CoreLib/Thread.cs,3980e012bae82e96) starts with a bunch of friendly constructors...

... which forward us into the scary `Thread.CoreCLR.cs` methods `Create` and `SetStartHelper`

```csharp
private void Create(ThreadStart start) =>
    SetStartHelper((Delegate)start, 0); // 0 will setup Thread with default stackSize

private void Create(ThreadStart start, int maxStackSize) =>
    SetStartHelper((Delegate)start, maxStackSize);

private void Create(ParameterizedThreadStart start) =>
    SetStartHelper((Delegate)start, 0);

private void Create(ParameterizedThreadStart start, int maxStackSize) =>
    SetStartHelper((Delegate)start, maxStackSize);
```

We cast both types to `Delegate` just to do a type switch later...

```csharp
private void SetStartHelper(Delegate start, int maxStackSize)
{
    Debug.Assert(maxStackSize >= 0);

    var helper = new ThreadHelper(start);
    if (start is ThreadStart)
    {
        SetStart(new ThreadStart(helper.ThreadStart), maxStackSize);
    }
    else
    {
        SetStart(new ParameterizedThreadStart(helper.ThreadStart), maxStackSize);
    }
}
```

... to call the same method from both `if` branches. At this point, I can hear Uncle Bob screaming.
```csharp
/// <summary>Sets the IThreadable interface for the thread. Assumes that start != null.</summary>
[MethodImpl(MethodImplOptions.InternalCall)]
private extern void SetStart(Delegate start, int maxStackSize);
```

But it's an `extern` method. Now what? Where is the code? We lost the trail.

*We must go deeper.*
<br/><iframe src="https://giphy.com/embed/7pHTiZYbAoq40" width="480" height="332" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/inception-7pHTiZYbAoq40">via GIPHY</a></p>

Let's head over to [GitHub](https://github.com/dotnet/runtime) and find out. We'll find the mapping of internal calls in [vm/ecalllist.h](https://github.com/dotnet/runtime/blob/e1ffadd6521b29db350e701b11d78a286ecd783a/src/coreclr/src/vm/ecalllist.h)

```cpp
FCFuncStart(gThreadFuncs)
// [...]
    FCFuncElement("SetStart", ThreadNative::SetStart)
// [...]
FCFuncEnd()
```

[Here](https://github.com/dotnet/runtime/blob/e1664815243da8eeb66e6f4e6e5a1468b8d9862d/src/coreclr/src/vm/comsynchronizable.cpp#L712) it is. Remember, we are still looking for the stack allocation code, and look at the `iRequestedStackSize` in our method signature, it looks like we are on the right track. Nice!

```cpp
FCIMPL3(void, ThreadNative::SetStart, ThreadBaseObject* pThisUNSAFE, Object* pDelegateUNSAFE, INT32 iRequestedStackSize)
{
// [...]
        unstarted->RequestedThreadStackSize(iRequestedStackSize);
// [...]
}
FCIMPLEND
```

In just a [few](https://github.com/dotnet/runtime/blob/e1ffadd6521b29db350e701b11d78a286ecd783a/src/coreclr/src/vm/threads.h#L4412) steps we arrive at the [treasure trove](https://github.com/dotnet/runtime/blob/e1ffadd6521b29db350e701b11d78a286ecd783a/src/coreclr/src/vm/threads.cpp#L2055):

```cpp
// We don't want ::CreateThread() calls scattered throughout the source.  So gather
// them all here.
```
Very good, I would not want them either.
```cpp
BOOL Thread::CreateNewThread(SIZE_T stackSize, LPTHREAD_START_ROUTINE start, void *args, LPCWSTR pName)
{
    CONTRACTL {
        NOTHROW;
        GC_TRIGGERS;
    }
    CONTRACTL_END;
    BOOL bRet;

    //This assert is here to prevent a bug in the future
    //  CreateTask currently takes a DWORD and we will downcast
    //  if that interface changes to take a SIZE_T this Assert needs to be removed.
    //
    _ASSERTE(stackSize <= 0xFFFFFFFF);

#ifndef TARGET_UNIX
    HandleHolder token;
    BOOL bReverted = FALSE;
    bRet = RevertIfImpersonated(&bReverted, &token);
    if (bRet != TRUE)
        return bRet;
#endif // !TARGET_UNIX

    m_StateNC = (ThreadStateNoConcurrency)((ULONG)m_StateNC | TSNC_CLRCreatedThread);
    bRet = CreateNewOSThread(stackSize, start, args);
```
༼ つ ◕_◕ ༽つ Look! It creates a new OS thread!
```cpp
#ifndef TARGET_UNIX
    UndoRevert(bReverted, token);
#endif // !TARGET_UNIX
    if (pName != NULL)
        SetThreadName(m_ThreadHandle, pName);

    return bRet;
}
```

The treasure must be very close by now!
```cpp
BOOL Thread::CreateNewOSThread(SIZE_T sizeToCommitOrReserve, LPTHREAD_START_ROUTINE start, void *args)
{
// [..]
#ifdef TARGET_UNIX
    h = ::PAL_CreateThread64(NULL     /*=SECURITY_ATTRIBUTES*/,
#else
    h = ::CreateThread(      NULL     /*=SECURITY_ATTRIBUTES*/,
#endif
// [..]
}
```

There it is! We arrived at the Windows API function `CreateThread` which will allocate `dwStackSize` (plus some rounding) bytes for the new stack:

```cpp
HANDLE CreateThread(
  LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  SIZE_T                  dwStackSize,
  LPTHREAD_START_ROUTINE  lpStartAddress,
  __drv_aliasesMem LPVOID lpParameter,
  DWORD                   dwCreationFlags,
  LPDWORD                 lpThreadId
);
```

And OS threads are unaware of our heap, so they won't allocate anything from managed heap, they will allocate available memory from the OS, or fail if they could not.

As .Net Core is cross-platform, for the sake of completeness, the UNIX [variant](https://github.com/dotnet/runtime/blob/84ec25f8034b2072838e516849313b6c23f409cd/src/coreclr/src/pal/src/thread/thread.cpp#L470) is way more complicated with lot of conditional compiling (and `goto`s!), if you scroll down a few pages, you can see it sets desired stack size using a [call](https://github.com/dotnet/runtime/blob/84ec25f8034b2072838e516849313b6c23f409cd/src/coreclr/src/pal/src/thread/thread.cpp#L670) to `pthread_attr_setstacksize`, this is all we wanted to know this time.

Well, it was a long but rewarding trip inside the belly of the beast. Wasn't it?

## So, the Correct Answer Is...

> I don't care.

It's just a stack, and as long as it works according to the standard, let the runtime put it wherever it wants ¯\\\_(ツ)\_/¯

## Resources

- [.Net Source Browser](https://source.dot.net/)
- [.Net Runtime project on GitHub](https://github.com/dotnet/runtime/)
- [CreateThread function](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread)
- [Thread Stack Size](https://docs.microsoft.com/en-us/windows/win32/procthread/thread-stack-size)
- [ECMA-335 Standard](http://www.ecma-international.org/publications/standards/Ecma-335.htm)
- [JVM specification, Java SE 14](https://docs.oracle.com/javase/specs/jvms/se14/html/jvms-2.html)
