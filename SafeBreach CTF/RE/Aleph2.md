
# Aleph2
Yet again, we are given a binary. This time it seems like an ELF binary, but when trying to execute it we get a segfault:

    root@beer:~/rev# file r2
	r2: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), statically linked, no section header
	root@beer:~/rev# ./r2
	Segmentation fault

My first guess was this file is packed. This is really common in CTF-ish reversing challenges. After running `strings` on the binary, I could see this string: 
`$Info: This file is packed with the AAA executable packer http://upx.sf.net $`

So this is an actually a standard packer, UPX. But it seems like the binary was modified to slightly misdirect us. `UPX` was replaced with `AAA`, but the URL remained true to the original.
Let's try to decompress the binary:

    root@beer:~/rev# upx -d r2
	                       Ultimate Packer for eXecutables
	                          Copyright (C) 1996 - 2018
	UPX 3.95        Markus Oberhumer, Laszlo Molnar & John Reiser   Aug 26th 2018

	        File size         Ratio      Format      Name
	   --------------------   ------   -----------   -----------
	upx: r2: NotPackedException: not packed by UPX

	Unpacked 0 files.
Hmm, strange. Not packed with UPX? Obviously it is. So this explains why `UPX` was replaced with `AAA`. The purpose was to slow us down a little bit. Anyway, a quick `sed` command to replace `AAA` with `UPX` allows us to fix the binary :)

    root@beer:~/rev# sed -i -- 's/AAA/UPX/g' r2
	root@beer:~/rev# upx -d r2
	                       Ultimate Packer for eXecutables
	                          Copyright (C) 1996 - 2018
	UPX 3.95        Markus Oberhumer, Laszlo Molnar & John Reiser   Aug 26th 2018

	        File size         Ratio      Format      Name
	   --------------------   ------   -----------   -----------
	     68096 <-     22996   33.77%   linux/amd64   r2

	Unpacked 1 file.

Executing the code is really similar to the previous RE challenge, only the flag length is printed. But I don't want the length, I want the flag.

    root@beer:~/rev# ./r2
	37

After running `strings` on the decompressed binary I can see a lot of strings referencing the Python library. So this is actually just Python code dynamically linked into an ELF binary. Running `ltrace` on the binary shows the Python calls. Here is the len() call on the list, for example:

    PyObject_Size(0x7f7f21eb0108, 18, 2, 0x438bb59acecb4f8d) = 37

If the code is similar to the previous challenge, then the object is actually a list and not a str. Therefore extracting it directly from the memory would be quite harder than just dumping the memory and looking for the string. Every object in that list is a different `PyObject` in itself...
After trying some things without success, I decided I will hook `PyObject_Size` to get the `PyObject` of the list and then extract the characters of the flag from there.

This is the hook code I ended up with:

    #include <Python.h>
	Py_ssize_t PyObject_Size(PyObject* o) {
	    Py_ssize_t size = PyList_Size(o);
	    for(int i = 0; i < size; ++i) {
	        PyObject* item = PyList_GetItem(o, i);
	        if(PyUnicode_Check(item)) {
	                PyObject* byte = PyUnicode_AsEncodedString(item, "UTF-8", "strict");
	                if(byte != NULL) {
	                 printf("%s", PyBytes_AS_STRING(byte));
	                } else {
	                 printf("Byte was null\n");
	                }
	        } else {
	                printf("Failed unicode check\n");
	        }
	    }
	    return 1337;
	}

This code iterates on each item in the list, checks if it's unicode-compatible (which should be), then encodes it as UTF-8 and prints it.
Now we need to compile the .so file and add it to the `LD_PRELOAD` environment variable when executing the binary:

    root@beer:~/rev# gcc -I /usr/include/python3.7/ -shared -fPIC hook.c -o hook.so
	root@beer:~/rev# LD_PRELOAD=./hook.so ./r2
	1337
	CTF{bf073zdac2b4746b727d762ac75fb607}
