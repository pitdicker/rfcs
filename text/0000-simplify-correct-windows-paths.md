- Feature Name: simplify_correct_windows_paths
- Start Date: 2016-01-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)


    TODO: what does the path module think about volume-relative paths?
    TODO: does is make sense to future-proof for NT paths? NO
    TODO: what are the limits of relative symlinks?
    TODO: convert canonicalize? -> maybe

# Summary
[summary]: #summary

On Windows transparently convert paths to use the Win32 File Namespace. This allows Rust programs to
open all paths that can exist on the underlying filesystem. Without a namespace we opt into a
compatibility layer that emulates DOS functionality, but limits paths to a maximum length of about
260 characters, and is unable to open paths with certain names.

# Motivation
[motivation]: #motivation

In most cases the 'correct' way to handle paths is to be able to work with any path that can exist
on the filesystem. Supporting legacy DOS functionality is often unnecessary, and can even be a
source of bugs for cross-platform applications.

Windows supports at least five kinds of paths. What just about everyone uses is what this RFC will
call Windows 95-style paths, which are a superset of MS-DOS-style paths. Since Windows NT there are
also three kinds of namespaced paths. The _Win32 File Namespace_ is of interest to this RFC.

Windows 95-style paths keep compatibility with DOS functionality and thereby create ambiguous
situations, and restrict paths to a maximum length of 259 characters. They are converted under the
hood by Windows to the Win32 File Namespace.

On namespaced paths Windows does conversions at all. There can be no ambiguity, so any path that can
exist in the filesystem can be opened. But this loses some useful functionality, like opening
relative paths and resolving path navigation like `..\`.

In practice only very few applications go through the trouble of using namespaced paths, mostly
backup software and other applications that intimately deal with the filesystem. All other
applications are broken when confronted with paths that contain unfortunate names or that are to
long. This includes rustc and cargo.

What we want is to not let Windows convert Windows 95-style paths to the Win32 File Namespace, but
do it ourself. We should only resolve relative paths, convert slashes and do path navigation. But
also do these conversions on namespaced paths for consistency.

Not automatically converting paths to their namespaced variant can be seen as the wrong default.
Currently Windows and the Rust standard library make DOS-compatibilty easy, and the ability to open
any path that can exist in the filesystem hard. This RFC proposes to reverse the situation.

Finally this change may benefit potential future filesystem functionality. Current best practices
for race-free filesystem operations recommend reusing file descriptors / handles as much as
possible, and to avoid absolute paths. Posix 2008 added function like `openat()` and other `*at()`
function for this. Windows has very good support for this also, but only with the `Nt*` functions,
not at the Win32 level. And at the NT level there is no support at all for DOS-specific
functionality.

# Detailed design
[design]: #detailed-design

Rust only uses the Unicode variants of WinAPI functions. Right before calling such a function the
path is converted from our variant of UTF-8 in `OsString`s to a 16-bit encoding. It is at this point
that we should convert paths to the Win32 File Namespace.

Doing any normalizations on paths is tricky, and something that has not always gone well with Rust
in the past. Because operating systems and libraries can all have their own (possibly non-sensical)
expectations about paths they should generally be touched as little as possible. But because we do
this conversion only right above the WinAPI level there is no risk. Converted paths do not leak out,
and we can specialize the conversion for the few functions that offer some custom functionality.
Which is currently only opening raw devices with `CreateFile` and creating relative symlinks.

## Converting paths

Thanks to two restriction Windows places on all paths, we can implement a conversion layer that does
not make any valid path impossible. First the characters `\`, `/` and 0x00 can never occur in a path
components (and are the [only ones](https://msdn.microsoft.com/en-us/library/ff470023.aspx). Second
is that although Windows does not natively support `.` and `..`, they are reserved as
[dot directory names](https://msdn.microsoft.com/en-us/library/ee381481.aspx). So we can safely
normalize a path with forward slashes and path navigation without making any path impossible to
access.

### 1. Convert forward slashes to backward slashes.
Just as it says, convert `/` to `\`.

### 2. Add a namespace
* If a path starts with `\\.\` it is a path in the Win32 Device Namespace.
  Return [error]. TODO
* If a path starts with `\\?\` it is a path in the Win32 File Namespace.
  Good, nothing to do.
* If none of the above, we are dealing with a Windows 95-style path.
  * If a path starts with `\\` it is an UNC path. Replace the `\\` with `\\?\UNC\`
  * If a path starts with `C:\` it is an absolute path. Here `C` can be any letter from a..z and
    A..Z. Prefix the path with `\\?\`.
  * If a path starts with `\` it is a rooted relative path. Join it with the root of the current
    working directory. TODO: namespace
  * This is a relative path. Join it to the path of the current working directory. Note that path
    will also need to have a namespace added.

### 3. Resolve `.\`, `\\` and `..\`
* Do not resolve in the paths namespace or prefix (like `C:\` or `\\server\share`)
* Strip empty path components `\\` and current directory components `.\`
* Strip parent path components `..\` and pop the component before it, if there is one.

## New methods

A method for opening raw devices with the Win32 Device Namespace:
```rust
mod os::windows::fs {
    pub trait OpenOptionsExt {
        fn open_device<P: AsRef<Path>>(&self, path: P) -> io::Result<File>;
    }
}
```

To help dealing with foreign libraries we should expose the same conversion functions as used in the
standard library for FFI.
```rust
mod os::windows::ffi {
    pub trait OsStrExt {
        fn needs_namespace(&self) -> bool; // is the same as is_windows_nice
        fn encode_path(&self) -> io::Result<Vec<u16>>; // TODO: normal result
        // will always convert slashes.
        // error if contains NUL
        // error if Win32 Device Namespace, NT Namespace or rooted relative path
        // namespace: add, absolute, keep+normalize, keep+slashes, drop
        // TODO: manually use is_absolute
        // TODO: how to recognise rooted relative paths?

        fn to_ansi_path(&self) -> io::Result<Vec<u8>>;
        // Error if not ASCII. TODO: maybe convert to ANSI?
        // Error if contains NUL
        // error if Win32 Device Namespace, NT Namespace or rooted relative path
        // removes namespace
        // error if reserved filename, trailing dots and slashes
        // path is to long
        fn to_wide_path(&self) -> io::Result<Vec<u16>>;
        // Error if contains NUL
        // error if Win32 Device Namespace, NT Namespace or rooted relative path
        // adds namespace, resolves relative paths
        fn to_namespaced_path(&self) -> io::Result<String>;
    }
}
```
use Path::canonicalize(). uses namespaced paths, but the result will contain no symlinks.
TODO: make it work with paths that do not exist. --> adds namespace
somehow to_wide()

TODO: somehow expose

TODO TODO TODO
You can rarely expect an external library to be able to open any path the filesystems supports. It
would have to use the unicode-aware WinAPIs, add a namespace to the paths and do similar conversions
as described in this RFC. So when a rust library interfaces with an external library it will just
about always have to provide some workarounds for paths (which is already true now to some extend).

TODO: Being nice (or, do not create hard to open paths)

- `is_namespace_needed`
- use `is_ascii`
- use `is_absolute`
- convert to wide with options: [add|remove|leave-alone] namespace, normalize.
- `is_windows_valid` and `is_windows_nice`

`is_windows_nice`:
    path length
    trailing dots and spaces
    reserved names
    is_windows_valid

- Library only supports ANSI-paths.
    no NULs, no unicode, drop_namespace (Cow), is_windows_nice
- Library supports Unicode-paths, but does no transparent conversion
    no NULs, encode_wide, max 260 chars, is_windows_nice OR
    no NULs, (optionally) add namespace
- Library expects UTF-8, converts to Unicode itself.
    TODO
- Always optionally do `is_windows_valid`

It is good to be able to access any path. But you are not really a good Windows citizen if you
create paths that most other applications can't open, and that horribly confuse Windows Explorer.



# Background

There is no single piece of documentation that describes in detail all the transformations that
Windows does on paths. Details necessary to evaluate whether the design of this RFC is correct, the
reasoning behind choices, what functionality will break an how to work around it. That is why this
RFC includes a rather large background section.

First a list of all the things that are now somehow treated specially by Windows, and how they are
supported by this RFC.

|        | type                          | support
|--------|-------------------------------|------------------------
| **Namespaces**                        ||
| (none) | Windows 95 paths              | yes
| `\\?\` | Win32 File Namespace          | yes
| `\\.\` | Win32 Device Namespace        | no (error)
| `\`    | NT Namespace                  | no (= rooted relative path)
| **Prefixes**                          ||
| `C:\`  | absolute path (driver letter) | yes
| `\\`   | absolute path (UNC)           | only with no namespace
| `UNC\` | absolute path (UNC)           | only with `\\?\` namespace
| `C:`   | volume relative path          | no
| `\`    | rooted relative path          | only with no namespace
| (none) | relative path                 | only with no namespace
| **Path name**                         ||
|        | convert `/` to `\`            | yes
|        | resolve `.\ `, `\\` and `..\` | yes
|        | trim dots and spaces          | no
|        | open DOS devices              | no
|        | limit path length             | no


## Namespaces and different kinds of paths

### Windows 95 paths

MS-DOS style paths come with the most restrictions. Filenames have to be short, ASCII (or some
8-bit codepage), and should not contain any character that can confuse the parsing of command line
arguments. They also support some useful tricks, like several kinds of relative paths, and resolving
`..\` and `.\`. It is also easy to open some devices like a serial or parallel port if a filename
matches a reserved name.

Windows 95 lifts a little bit of these restrictions, by allowing long file names (up to 255
characters), spaces in file and directory names, and Unicode. But most of the tricks and
restrictions remain the same, for backwards compatibility. Notably also the maximum path length of
259 characters.

Any DOS or Windows 95-style path name that get passed to a Windows API is put through a conversion
function in ntdll.dll (`RtlDosPathNameToNtPathName`). It is responsible for emulating all the legacy
functionality. It performs the following conversions:

- add a namespace if the path not already has one
- make paths relative to the current working directory absolute
- open DOS devices like `CON` (console) and `AUX` (serial device)
- path navigation by resolving `.\ `and `..\` and removing double slashes
- conversion of forward slashes to backward slashes
- trimming dots and spaces from the end of path components
- limit the path length to 259 characters

### Win32 File Namespace paths

- `\\?\` **Win32 File Namespace**.
  All paths that refer to files or directories on the filesystem.

Currently namespaced paths are special cased in the standard library to not have any conversions be
done on them. But when we start to transparently convert between normal and namespaced paths this
implies that we need to treat both kinds of paths in the same way. Otherwise we get surprising
behavior (and we already have now). For example, suppose we join a relative path that uses forward
slashes (as is common in cross-platform code) to some base path. If the base path does not have a
namespace the slashes would be converted, and everything works fine 99,9% of the time. But if the
base path has a namespace it would break, with the slashes not being converted.

Namespaced paths can only be absolute, not relative. And they don't support forward slashes and
Unix-style path navigation using `.\` and `..\`. We wish to support this functionality and implement
our own support for it.

Currently paths prefixed with `\\?\` are special cased to not have any conversions done on them.
But because `/`, `.\`, `..\` and `\\` can not exist in valid _Win32 File Namespace_ paths anyway,
they remain unmodified. By not special-casing them we support the use case where a relative
Unix-style path (as is common in cross-platform code) is joined to a namespaced path:
`\\?\C:\foo\bar\/../baz/qux`. Otherwise code will work always when the base path is not namespaced,
but will break silently when it is.


### Win32 Device Namespace paths

Examples:
* `\\.\CON` (console device)
* `\\.\C:\dir\file.txt` (`\dir\file.txt` is passed to the driver of the `C:` device)

Paths in the Win32 Device Namespace do not refer to the filesystem, but to devices placed in a
special directory in the [Object Manager](http://www.osronline.com/article.cfm?article=381).
Everything the path contains after the device it points to, is parsed by the device driver. Because
these drivers can have their own restrictions and requirements, we should not do any conversations
on these paths.

`CreateFile` seems to be the only WinAPI function that can work with Win32 Device Namespace paths.

If we were to support Win32 Device Namespace paths, we would have to special-case them so we do not
do any conversions on them. These paths can cause subtle bugs: they can not be used for any
function that does not use `CreateFile`, and a Device Namespace path breaks a relative path when it
is used as the base.

Instead this RFC choses to always deny Device Namespace paths. This will keep our normal path
handling code simple and consistent. But to restore the missing functionality we should add a new
method `open_device(path)` to `OpenOptions`. This method will only accept paths with the `\\.\`
namespace, on not do any conversions on them.


### NT Namespace paths

Examples:
* `\TODO`
* `\Device\HarddiskVolume0\dir\file.txt` (`\dir\file.txt` is passed to the driver of
  `\Device\HarddiskVolume0`)
* `\??\C:\dir\file.txt` (`C:` is a symlink, probably to `\Device\HarddiskVolume0`)

NT Namespace paths are used by the internal `Nt*` en `Zw*` functions of Windows. Normal WinAPI
functions can not handle them. These paths are used to navigate in the Object Manager. Everything
the path contains after the device it points to, is parsed by the device driver. There is no
question about whether we should support NT namespace paths in general, as we simply can't. The
standard library uses normal WinAPI functions. Instead we treat them as rooted relative paths, with
which they are ambiguous.

Of special interest are paths that start with `\??\`. When the two Win32 Namespaces are converted
down to NT internal paths, they end up in this directory in the Object Manager (not completely true
as `??` does not actually exist. It is hard-coded to look first in a local and then a global
directory, to support Terminal Sessions and Run as...). At least some WinAPI functions allow an NT
namespace to this directory to pass through. This does however not offer any functionality that is
not already available through the Win32 Namespaces. If we were to `\??\` paths, no normalization
should be done on them for the same reasons as Win32 Device Namespace paths.

Generally speaking NT Namespace paths should not be mixed with normal filesystem paths. Any code
that somehow obtains one by calling internal functions should only use it with internal functions,
and not pass it to the standard library. Because of this and because `\??\` paths would need to be
special-cased to have no conversions be done on them, this RFC proposes to not support them.

If the NT Namespace path starts with `\??\` it can always be converted to `\\.\`, and possibly to
`\\?\`. But if passed to the standard library we will treat it as a rooted relative path. It will
just about always be invalid, as question marks are invalid characters in a path name.

## Absolute and relative paths

### Absolute paths

Examples:
* `C:\path` absolute path to a local drive
* `\\host\share\path` absolute path to a network share (UNC) with Windows 95-style paths
* `\\?\UNC\host\share\path` absolute path to a network share (UNC) with Win32 File Namespace paths

An absolute path can be on either a local drive or a network share. Paths on a local drive start
with a letter (disk designator), a colon, and a slash. `C:\` is not ambiguous with alternate
datastreams, because directories can't have alternate datastreams.

Paths to network shares always start with two slashes for Windows 95-style paths, and with `UNC\`
for Win32 File Namespace paths. They follow the
[Universal Naming Convention (UNC)](https://msdn.microsoft.com/en-us/library/gg465305.aspx). An UNC
path always has to contain a host and a share. Host names are complicated but at least `.` and `?`
are not allowed, creating no ambiguity with Win32 Namespaces. Some sources say a dot as host name
refers to the local machine, but this is not supported by Windows.

### Relative paths

Examples (where the current directory is `E:\baz\qux\`):
* `path` becomes `\\?\E:\baz\qux\path`
* `.\path` becomes `\\?\E:\baz\qux\path`
* `\path` becomes `\\?\E:\path`
* `\path` becomes `\\?\UNC\host\share\foo\bar` if the current directory is `\\host\share\baz\qux\`.

Commonly relative paths are joined to some base path they are relative to before calling a
filesystem function. But if a path is still relative when passed to the Windows API, it gets
resolved against the current directory.

Of interest is that Microsoft now
[says](https://msdn.microsoft.com/en-us/library/windows/desktop/aa364934%28v=vs.85%29.aspx) that
_"multithreaded applications and shared library code should (...) avoid using relative path names.
The current directory state is stored as a global variable in each process, therefore multithreaded
applications cannot reliably use this value without possible data corruption from other threads that
may also be reading or setting this value. (...) Using relative path names in multithreaded
applications or shared library code can yield unpredictable results and is not supported."_

But using relative paths is not uncommon, especially on Unix. And not implementing them is probably
a big breaking change. So this RFC proposes to support them. Because the normalization of relative
paths is now under our control, we might be able to improve upon the thread-safety concerns in the
future. For example by having a shadow copy of the current directory in a static `RwLock`, that is
not available to foreign libraries (which a well-behaving library should not need anyway).
TODO: is this possible?

A Windows-only 'trick' are rooted relative paths, paths that start with a slash. Such a paths is
relative to the disk drive or UNC network share of some base path. This way the path is almost but
not entirely absolute. It is only rarely useful, for example to work with paths in some fixed
directory structure but with an unknown drive letter, like on a cd-rom or floppy disk. Being
relative to the root of the current directory has even less uses. But rooted relative paths are well
supported by the `Path` module, natively supported by Windows for symlinks, and create no ambiguity.
So it makes sense to support them, even if only for consistency.

Resolving relative paths is as simple as joining them with the current directory. Or in the case of
rooted relative paths joining them with the root of the current directory.
TODO: or open relative to HANDLE of current dir.

### Volume relative paths

Examples:
* `E:foo\bar` can become `E:\baz\qux\foo\bar` if the current directory is `E:\baz\qux\`.
* `E:foo\bar` can become `E:\foo\bar` if the current directory is not on `E:\`.
* `E:foo\bar` can become `E:\baz\qux\foo\bar` if the current directory is not on `E:\`, but we
  use a hack with environment variables that remembers a previous current directory for the drive.

Volume-relative paths where a feature on DOS 2.0 (see
[The Old New Thing](https://blogs.msdn.microsoft.com/oldnewthing/20101011-00/?p=12563) for some
background). This type of path is relative to the current working directory _of the current volume_.

Windows NT+ still somewhat supports volume-relative paths, but not reliably. This is because Windows
no longer stores the current directory for every volume anymore, but only one current directory.
So it works only if the current directory happens to be on the same volume as the volume-relative
path. Volume relative paths have never worked for network locations.

The Visual C Runtime hacks around the missing functionality by storing the path of the current
directory for every drive in
[environment variables](https://blogs.msdn.microsoft.com/oldnewthing/20100506-00/?p=14133/).
The name of such variables is of the form `=C:`. WinAPI does take these variables into account
when the volume of the normal current directory does not match. But because only applications that
use MVC's `_chdir()` set the environment variables (and Rust curently does not set them either)
volume-relative path remain unreliable.

Also volume relative paths are ambiguous with alternate datastreams (although it becomes a bit
pathological). For example: `E:stream` can not only expand to `E:\baz\qux\foo`, but also to
`E:\baz\qux\E:stream`, the alternate datastream `stream` of `E:\baz\qux\E`.

Because volume relative paths are ambiguous, rarely used and unreliable, the RFC proposes not to
emulate them.

## Conversions on the path name

### Converting `/` to `\`

At the Win32 level Windows converts any forward slash to a backward slash. It does not matter if it
is part of the namespace, disk letter, UNC prefix, or path name. But at the deeper NT level only
backward slashes are valid. Only rarely it becomes visible `/` is not native, like when creating a
symlink (because the path gets interpreted by the kernel when the link is accessed, not by the Win32
API when creating it).

The Rust standard library uses almost always the Win32 API, so converting slashes should almost
never be necessary on our side. But it basically comes for free with the resolving of path
components, so we should just convert them instead of creating more special cases.

That both slashes are valid means we have to be careful to recognise prefixes correctly. For example
`\\?\`, `//?/` and `\/?\` all indicate a Win32 File Namespace path.

### Resolving `.\ `, `\\` and `..\`

We only should not do the conversion on the paths namespace and prefix. Otherwise
`\\?\C:\` and `\\server\share\dir` would not work anymore.

We wish to support paths that contain `.` and `..` to indicate the current directory and the parent
directory, and also to support empty path components. Contrary to Unix, Windows APIs do not seem to
assign any special meaning to paths that start with `.\`, so we can safely drop them from the path.
Also collapsing multiple slashes into a single one is ok, an empty path components make no sense
from a filesystem perspective.

Correctly resolving a `..` path component is more involved than simple string manipulation. If
the parent path component is a symlink we would have to lookup the real location of the symlink
in the filesystem. For example `C:\a\b\..\` may refer to `C:\a\`, or to `C:\something_else`,
depending on whether `b` is a symlink or not.

Steps to correctly resolve a path (after joining relative paths with a base path and converting
forward slashes):
1. ignore double slashes and `.\`'s.
2. work from left to right.
3. get the real path of the path component right before a chain of `..`'s.
4. if the path exists pop as many path components from the real path as there are `..`'s following.
5. if the path does not exist drop the last path component and the first `..` from the chain.
6. do not navigate past the root directory.
7. cache up to which point we now the path exists in the filesystem, so we don't have to lookups up to there anymore.

for each path component right before a chain of `..`'s.
try opening without following reparse points
    if normal file/dir -> get real path.
    if reparse point -> try opening following reparse points
        if exists -> get real path
        else -> read reparse point; manually resolve if relative. fail if > 63, as we are probably in a cycle.
    else -> pop the last path component and the first `..`

TODO: write some code.

TODO: it is not trivial _at all_.
Having to query the filesystem to resolve the path brings a lot more complexity. We have a
slight performance impact because we possibly have do more system calls (although limited,
every chain of `..\..` needs just one, and normals paths don't use them all that much). Can
we be sure a function remains sort-of atomic? Would it be possible to pass a crafted path to
`open()` to DOS the system? We have to make sure resolving happens at the right time (the last
possible moment) to account for the possibility that the filesystem may change in the meantime.
Do we have to be careful to not get into an infinite loop if symlinks form a cycle?

This RFC proposes to _not_ correctly resolve paths given the added complexity. Symlinks are rare
on Windows, as it is very hard to create them. And even when a path contains symlinks, navigating
past it with a parent path components is an edge case. Also there is precedent for this, as
Windows itself does path resolution symlink-unaware. And no one else (like Cygwin) seem to do it
strictly correct either.

But we should keep our options open to do it correct in the future. If in the future Windows changes
to support symlink-aware path navigation or we have confidence in doing it in the standard library.


### Trimming of dots and spaces from path segments

Dots and spaces (and only these two characters) are silently trimmed from the end of all path
components. For example `C:\morse . .. \foo` becomes `C:\morse\foo`.


### DOS devices / reserved names

With namespaced paths there is a clear distinction between paths that can exist on the filesystem
and paths that point to a device. For DOS compatibility filesystem paths containing some reserved
names are automatically recognised to point to DOS Devices.

Windows recognizes a path to be a device if the last path component matches `CONIN$`, `CONOUT$`,
`CON`, `NUL`, `AUX`, `PRN`, `COMn` or `LPTn` (where  `n`=1..9). The path may not contain a trailing
slash. Matching is case insensitive, and any trailing dots and spaces are ignored. Finally it only
checks the filename, so any alternative datastream is ignored.

As an example all these paths open a console: `CON`, `con`, `C:\CON`, `C:\CON:stream`, `\CON`,
`C:CON`, `CON . ..` and `C:\CON . ..:stream`. These do not: `BACON`, `CON\`.

To emulate this functionality, is is enough to put the recognized device name in a
_Win32 Device Namespace_ prefix. So the examples before all become `\\.\CON`.
TODO: and use `open_device()`



### Special case: symlink and junction targets
Symlinks can be either absolute or relative. Naively making the path absolute will break relative
symlinks. So if we create a symlink with an absolute path, we should do the conversions described
above. This is also true for Junctions, which are almost but not entirely the same as directory
symlinks, and only accept absolute paths.

Windows supports resolving symlinks with normal relative paths and root-relative paths (where they
are relative to the path of the symlink, not the current working directory). Also supported are `.`,
`..` and double slashes. They get resolved on the path of the symlinks target. So resolving the
first chain of `..`'s is done correctly, but all others are still symlink-unaware. The only
conversion we should do on relative paths ourself is converting `/` to `\`.

Volume-relative paths are
[not really](https://msdn.microsoft.com/nl-nl/library/windows/desktop/aa363866%28v=vs.85%29.aspx)
supported, being relative to the current directory of a drive really does not make sense for a
symlink. We should not accept them, just as we should not for normal paths. Not special-casing them
means volume-relative paths end up as an invalid link to some impossible relative path.

A second problem with symlinks is that when we create them with `CreateSymbolicLink` adding a
namespace is not entirely transparent. A user can inspect the string that makes up the symlink, and
will see we converted the path. Exactly for this reason Windows symlinks support both a 'substitute
name' and a 'print name'. The substitute name is a properly normalized path in the NT Namespace
`\??\`. The print name should be a nice user-facing variant of the path, in our case the normalized
path but without a namespace. The function `CreateSymbolicLink` is not able to set both names, but
if we create a symlink through `DeviceIoControl` we can.


## Functionality that remains unchanged

Some functionality of Windows filesystems is implemented at a deeper level than the distinction
between normal and namespaced paths. As such they do not change because when we transparently add a
namespace to a path. They are listed here to help evaluating this RFC really does not break them.

**DOS 8.3-style names**. Short filenames are a feature of the filesystem (they exists as a special
kind of hard links, see this [paper](http://www.osdever.net/documents/LongFileName.pdf) page 8).
Whether we use a namespaced path or not does not change how they work. Application should however
not rely on them being available, as the can be
[disabled](https://technet.microsoft.com/en-us/library/ff621566%28v=ws.10%29.aspx).

**Invalid characters**. The following characters can not exist in the name of a file or directory:
`\`, `/`, `:`, `>`, `<`, `|`, `?`, `*`, and the characters 0x00...0x1F.

**NTFS alternate data streams**. The final path component can be of the
[form](https://msdn.microsoft.com/en-us/library/cc422524.aspx) `filename:streamname:streamtype`. The
name of the alternate stream can contain any kind of character except `\`, `/`, `:` and 0x00. In all
other path components the stream seperator `:` is not allowed. An empty stream name is possible, and
a stream type is optional.

**Maximum path component lengths**. Path components can never contain more than 255 characters. Note
that a character that is encoded with two UTF-16 surrogates counts as two. TODO: including stream name?

**Case (in)sensitivity**.
Windows treats paths by default case-insensively, but filesystems like NTFS can be case-sensitive.
This can cause applications to be unable to open a file if the path only differs in case from an
other file. But this is rare to encounter in practice as Windows does not allow case-sensitive
filesystem operations unless a registry key is modified.

Handling case sensitivity with paths is out of scope for this RFC, and probably also for the
standard library. In short `CreateFile` and `FindFirstFile` are the only functions that allow a flag
to indicate case-sensitive paths. All other filesystem operations have to be emulated by doing
operations on a handle obtained with `CreateFile`.

# Drawbacks

Increased complexity. --> NOT

We lose support for
- volume-relative paths
- NT-internal namespace `/??/`
- opening devices by using reserved names
- the silend trimming of dots and spaces from the end of path components

TODO: our path handling differs from most Windows programs. When is this a problem?
- for paths to devices --> does not make sense
- volume-relative paths
- ??


# Alternatives

- support device paths without a special function
  --> it would be possible to have a path that can be opened, but all other functions fail on.
- only add a namespace prefix if it is needed
  --> increases complexity, no advantages.
  --> easy to get wrong, see Java.
- support two kinds of paths, verbatim and legacy
  --> see Racket
- fully emulate Windows legacy paths, but without the length restriction
  --> only solves the length problem, but not the rest.


## Only add a namespace prefix if it is needed
This RFC proposes to always add a namespace to a path, but we could just as well only add a
namespace prefix if it is needed. This means we first have to determine if a path is to long or
contains components that are special to `RtlDosPathNameToNtPathName`. And we would still need to
perform the conversions in this RFC if the path does need a namespace. So this does not really gain
us anything, but adds some extra complexity.

[Java](http://www.docjar.com/html/api/sun/nio/fs/WindowsPath.java.html) does something like this. It
transparently adds `\\?\` if a path is to long. But because it does not handle any other differences
TODO =inconsistent

## Fully emulate Windows legacy paths, but without the length restriction
TODO

## Support two kinds of paths
* emulate more quirks (Racket)
- [Racket](https://docs.racket-lang.org/reference/windowspaths.html) contains a seemingly very
  complex implementation for Windows paths. Racket tries to emulate Windows legacy behavior, while
  implicitly converting paths to `\\?\` form as needed. TODO: and has its own extensions

[Racket](https://docs.racket-lang.org/reference/windowspaths.html) supports two kinds of paths.
If the path does not have a namespace it is emulated as path with legacy semantics, but without the
length restriction. If a path has a namespace, it is passed almost verbatim. Relative paths can opt
out of the legacy semantics by prepending a relative namespace: `\\?\REL` or `\\?\RED`.

An implementation like Racket adds a lot more complexity than this RFC. Also I think it makes the
wrong trade-off, with emulating legacy path behavior being the default instead of being able to open
any path.

# Unresolved questions

What parts of the design are still TBD?
FFI
