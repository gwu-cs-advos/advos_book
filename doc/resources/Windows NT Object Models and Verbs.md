Notes from Mr. Sean McBride about the Windows abstractions

# Scoping discussion of objects (from low-level to high-level)

NT "Executive Objects"
- A a common "abstract base type" is used by all types of objects.
- There is effectively a hierarchy of object models
- Other Executive Subsystems are responsible for specific object types. They register objects with object manager, register lifecycle hooks, and handle the state in the "object body"

Win32 Object
- The Personalities wrap the NT Executive Objects. Win32 typically wraps the NT Executive objects with Windows desktop stuff. Applications get a opaque handle to a win32 object.
- Technically this doesn't have to be object-based. The Windows POSIX subsystem did POSIX-y things.

COM Objects
- This is orthogonal to Win32 Objects or "NT Executive Objects."
  - Handles to Win32 Objects are used at the interface between a process and a Windows API.
  - COM is a binary interface for interprocess communication. The heritage is around early distributed system stuff: RPC, CORBA, etc.
- How can userspace services provide and use services, either locally or over the network?
- This provides a mechanism for native compiled userspace applications and libraries to register interfaces and implementations of interfaces. Produces and consumers are loosely coupled, and the invocation of a function results in a local procedure call or a remote procedure call.
- An example use-case is OLE (https://en.wikipedia.org/wiki/Object_Linking_and_Embedding) to allow an Excel Spreadsheet to be embedded in a Word document. The Word binary calls Excel via COM.
- DirectX also uses COM.

.NET Objects
- Higher-level objects built around the CLR. Bytecode rather than native code
- Auto-generates .NET wrappers around native COM: https://learn.microsoft.com/en-us/dotnet/standard/native-interop/cominterop
- Can also expose .NET as COM: https://learn.microsoft.com/en-us/dotnet/core/native-interop/expose-components-to-com

Metro / Modern / WINRT / "Windows Store style" / "Windows Runtime"
- Modern Sandboxed APIs with language projections. More secure, more locked down, etc.
- Use COM for inter-procedural support with traditional win32, .NET, etc.
- C++/WinRT interacts can consume COM: https://learn.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/consume-com
- C++/WinRT can be used to expose COM: https://learn.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/author-coclasses
- Rust/WinRT is probably equivalent: https://github.com/microsoft/windows-rs

# What is worth covering with students in Advanced OS?

I think the NT "Executive Objects" layer is the best part to focus on. https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/_kernel/

Normalized Functions:

The naming scheme is a carry-over from the VMS operating system.

{subsystem_prefix}{verb}{noun}

ObCloseHandle
Ob = Object Manger
Verb = Close
Noun = Handle

It might be worth showing that this is relatively consistent across the Microsoft object layers. Equivalent discussion for PowerShell:
- https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.3



# Optimal Resource that is a print book (~10x as good as all free alternatives)
- "Inside Windows NT" (1993), Chapter 3 (pages 49-81)
- I couldn't find a PDF of the book, but available for checkout at Internet Archive (not sure if this means one-student-at-a-time). https://archive.org/details/insidewindowsnt00cust
- This chapter uses terms defined in Chapter 2 (Executive vs. kernel, personality, etc.), so Chapter 2 might be be needed as well. Hard to understand the levels of abstraction without realizing the OS was intended to support the known universe circa 1988: OS/2, win32, 16-bit windows, MS-DOS, POSIX, etc.
- Related study questions from an OS course circa 1997
- http://matcscserver.uu.edu/classes/wilms/CSC425/1997%20Fall/book_rev.pdf

# Free alternatives
- https://en.wikipedia.org/wiki/Architecture_of_Windows_NT
- https://en.wikipedia.org/wiki/Object_Manager_(Windows)
- https://www.itprotoday.com/compute-engines/inside-nts-object-manager
- Video on Windows Object Manager: https://www.youtube.com/watch?v=tFSvtOrcwTA
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/object-based
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/managing-kernel-objects
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/managing-kernel-objects
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/object-names
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/object-directories
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/life-cycle-of-an-object
- https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/object-handles

# Kernel Object interaction from user space processes via win32 personality
- https://learn.microsoft.com/en-us/windows/win32/sysinfo/kernel-objects

# How one could demo:
- The Object tree isn't exposed via the Windows API, so typically only drivers deal with it directly.
There are tools to be able to poke at it though the WinObj tool. Could all be done in a VM.
    - https://learn.microsoft.com/en-us/sysinternals/downloads/winobj
    - https://github.com/hfiref0x/WinObjEx64

# Source Code Possibilities:
- Windows Research Kernel is a minimal NT kernel that Microsoft distributes for research and teaching (This is an illegal distribution, but I think you can legally request for course somehow) - https://github.com/HighSchoolSoftwareClub/Windows-Research-Kernel-WRK-/tree/master/WRK-v1.2/base/ntos/ob
- ReactOS Re-implementation - https://github.com/reactos/reactos/tree/master/ntoskrnl/ob
