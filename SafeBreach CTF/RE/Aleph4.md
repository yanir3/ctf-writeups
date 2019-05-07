# Aleph4
Another ELF. Let's do a simple initial analysis:

    root@beer:~/rev# file r4
	r4: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=28ba79c778f7402713aec6af319ee0fbaf3a8014, stripped
	root@beer:~/rev# readelf -h r4
	ELF Header:
	  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
	  Class:                             ELF64
	  Data:                              2's complement, little endian
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI Version:                       0
	  Type:                              EXEC (Executable file)
	  Machine:                           Advanced Micro Devices X86-64
	  Version:                           0x1
	  Entry point address:               0x401a75
	  Start of program headers:          64 (bytes into file)
	  Start of section headers:          5628512 (bytes into file)
	  Flags:                             0x0
	  Size of this header:               64 (bytes)
	  Size of program headers:           56 (bytes)
	  Number of program headers:         8
	  Size of section headers:           64 (bytes)
	  Number of section headers:         29
	  Section header string table index: 28

Seems pretty straightforward. Running this binary however, produces no output at all? Time to use `strace`/`ltrace` to figure out what it's doing.
`ltrace` produces interesting output, I noticed the binary uses some python-related files, and calls `fork()`, so it's important to add the `-f` flag to attach to child processes. After tracing child processes, we can notice the process is trying to get some environment variable and also calls dynamic linker functions:

    [pid 9914] getenv("_MEIPASS2")                   = nil
	.. (later) ..
	[pid 9907] dlsym(0x2526580, "PySys_AddWarnOption")                                                                                        = 0x7fa9a4110a80
	[pid 9907] dlsym(0x2526580, "PySys_SetArgvEx")                                                                                            = 0x7fa9a4110310
	[pid 9907] dlsym(0x2526580, "PySys_GetObject")                                                                                            = 0x7fa9a410d290
	[pid 9907] dlsym(0x2526580, "PySys_SetObject")                                                                                            = 0x7fa9a410d210
	[pid 9907] dlsym(0x2526580, "PySys_SetPath")                                                                                              = 0x7fa9a410fcb0
	[pid 9907] dlsym(0x2526580, "PyEval_EvalCode")                                                                                            = 0x7fa9a414a290
	[pid 9907] dlsym(0x2526580, "PyMarshal_ReadObjectFromString")                                                                             = 0x7fa9a4129d70
	[pid 9907] dlsym(0x2526580, "PyUnicode_FromString")                                                                                       = 0x7fa9a41b04a0
	[pid 9907] dlsym(0x2526580, "Py_DecodeLocale")                                                                                            = 0x7fa9a4102250
	[pid 9907] dlsym(0x2526580, "PyUnicode_FromFormat")                                                                                       = 0x7fa9a41b7320

After googling the `MEI_PASS2` env variable, I understood it's related to PyInstaller, which freezes Python application into executables. This gave me a good idea on how to go forward. `dlsym` is a function which returns the address for a symbol in a dynamic library (in this case the dynamic library was `libpython3.7m.so`.
For those unfamiliar with PyInstaller internals (I was clueless at the beginning too), from my understanding it basically compresses (with zlib) the Python libraries and adds them to the final binary, then it compiles your Python into bytecode and executes it through those libraries. This can now explain all those dynamic library calls. Let's move forward.

## Extracting the Python bytecode
So obviously whoever made this challenge wrote some Python code and it's just hiding there somewhere. How will we extract it? There is more than one way of course, you could use some PyInstaller unpacker, I tried, but none worked. So I wanted to do something more interesting.
Previously, we saw **PyMarshal_ReadObjectFromString** was resolved at runtime. For those who don't know, **marshal** is Python's internal method of serializing and deserializing Python objects. In this case, PyInstaller uses this function to deserialize the Python code object, then they pass the code object to `PyEval_EvalCode` to actually execute the code.
The signature for the function is:

    PyObject* PyMarshal_ReadObjectFromString(const char *data, Py_ssize_t len)

The idea is to hook this function call.  `*data` is the raw Python bytecode, we can simply save it and then figure out how to extract the flag.

## Writing the hook
Since the app uses `dlsym` to resolve the symbol from an open dynamic library handle, it's a bit more complex to hook this. We actually need to hook `dlsym` to return a pointer to our hook, and then resolve the symbol ourselves (so we can forward the function call to the real function). 

    #include <Python.h>

	PyObject* (*marshal)(const char*, Py_ssize_t);
	extern void *_dl_sym(void *, const char *, void *);

	PyObject* PyMarshal_ReadObjectFromString(const char *data, Py_ssize_t len)
	{
	   FILE *f = fopen("dump", "w");
	   fwrite(data, sizeof(char), len, f);
	   fclose(f);
	   printf("called (len: %d)\n", len);
	   return marshal(data, len);
	}


	extern void *dlsym(void *handle, const char *name)
	{
	    if (!strcmp(name,"PyMarshal_ReadObjectFromString")) {
	        printf("dlsym hook match! updating ptr\n");
	        marshal = _dl_sym(handle, "PyMarshal_ReadObjectFromString", dlsym);
	        return (void*)PyMarshal_ReadObjectFromString;
	    }
	    return _dl_sym(handle, name, dlsym);
	}

This code is pretty simple. It hooks `dlsym` and returns a pointer to our hooking function (when the requested symbol is what we want to hook), and also saves the real pointer in the `marshal` variable, so now our hooking function can intercept the calls without interrupting the usual execution of the program. This code will also dump the last function call to the `dump` file. Let's build and run:

	root@beer:~/rev# gcc -I /usr/include/python3.7/ -shared -fPIC hook2.c -o hook2.so; LD_PRELOAD=./hook2.so ./r4
	dlsym hook match! updating ptr
	called (len: 4157)
	called (len: 131440)

So we got two calls. I actually reviewed both calls, the first call is extra code from the PyInstaller package and isn't relevant here. The second code is what's interesting.
Since we have the raw serialized data, we can now execute it in our interpreter context and see what happens. It's important to note that this code was compiled with **Python 3.7**, this caused me to waste a lot of time initially because I was running 3.6.8, the Python bytecode sometimes changes from release to release so needless to say, this bytecode did not execute successfully on Python 3.6.8.
Executing the bytecode in your local interpreter is pretty straightforward:

    >>> import marshal
	>>> marshal.load(open("dump", "rb"))
	<code object <module> at 0x7fb121af2420, file "tmpkv3r95zl.py", line 1>
	>>> exec(_)

You can now look at the local variables from the script using `vars()`, and you can guess that the flag is in there somewhere. You should see this in the bottom of your vars output:

    ...oBHpxrfcFCLAuQGd at 0x7ff524002950>}, 'oBHpxrfcFCLAuQsh': {'STACK': [67, 84, 70, 123, 51, 33, 53, 97, 101, 51, 102, 55, 100, 98, 100, 49, 49, 101, 48, 53, 97, 53, 100, 68, 79, 95, 73, 84, 95, 89, 79, 85, 82, 83, 69, 76, 70, 125]}, 'oBHpxrfcFCLAuQsV': 21}

So this `_STACK` suspiciously looks like a string hidden in a list, let's extract it:

    >>> "".join([chr(c) for c in oBHpxrfcFCLAuQsh['STACK']])
	'CTF{3f5ae0fc67zcd14e05a5d8a6frr494a0}'
