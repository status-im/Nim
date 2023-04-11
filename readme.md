# Heaptrack for Nim

This branch allows you to use [heaptrack](https://github.com/KDE/heaptrack) on nim programs.

## Heaptrack
Heaptrack is a neat little tool used for RAM usage debugging, allowing to find leaks, allocation hotspots, etc.

It's usable in two ways:
- Preload: ex, `heaptrack ls`, will preload `libheaptrack_preload.so` into ls, and record every allocation during the entire program
- Inject: ex, `ls &; heaptrack -p $(pidof ls)`, will attach to the running program using `gdb`, inject `libheaptrack_inject.so`, and record every allocation from this point until the program finishes or `heaptrack` is killed.

Preload is more useful for short-running application, and allow to detect memory leaks as well.

Inject is useful to monitor a precise moment of a long running application, and can't detected memory leaks (since the program could `free` the allocated memory after we've detached, or have `alloc`ated before we attached)

Internally, `heaptrack` works by writing into a file each allocation and dellocation done by the program, along with the stacktrace for each allocation, eg:

```
+ 55c7e2ff17ea 15 #Allocated 21 bytes into address "55c7e2ff17ea"
- 55c7e2ff17ea #Deallocated pointer "55c7e2ff17ea"
```
(note, this is a simplification of the actual format, which is optimized to be machine-readable but not human readable)

For most of programs, it just wraps `malloc` and `free` into custom function to create this file, but obviously, nim doesn't use `malloc` and `free`, so some trickery is required.

## Heaptrack in Nim
This branch simply plugs the `libheaptrack`'s procs into the nim gc, allowing to use heaptrack for nim programs.
```bash
nim c -d:heaptracker test.nim
heaptrack ./test #works as any program
```

Be careful, to use the `inject` mode, the `libheaptrack_inject.so` must be used instead of `libheaptrack_preload.so` during compilation:
```bash
nim c -d:heaptracker -d:heaptracker_inject test.nim
LD_LIBRARY_PATH=/usr/local/lib/heaptrack ./test &
heaptrack -p $(pidof test)
```
The binary must use the same `.so` as `heaptrack`, so an `inject` binary won't work with `prelude`, and a `prelude` binary won't work with `inject`

Be also careful, once a binary has been compiled with `-d:heaptracker`, it won't run on a device where `heaptrack` is not installed.
For `inject`, you also need to specify the `LD_LIBRARY_PATH` to locate the `so` files.

## Heaptrack for Nimbus
To build nimbus with heaptrack enabled:
Add to the Makefile the required flags, as [shown here](https://github.com/status-im/nimbus-eth2/commit/90eca6d54309db669f03866fb8d432b739687f70)

Change to build Dockerfile to add `apt install heaptrack` and set the `LD_LIBRARY_PATH` properly

Add `gdb` capabilities to the docker-compose.yml
```yaml
    cap_add:
      - 'SYS_PTRACE'
    security_opt:
      - 'seccomp:unconfined'
```

Once everything is setup, you can go into the container, install gdb, and run `heaptrack -p 1` as usual.
When you're happy with your capture, stop heaptrack, retrieve the file on your computer for analysis, and voilà!