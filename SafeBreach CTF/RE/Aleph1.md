
# Aleph1
We're given a strange binary file, no signature or anything. Let's run some forensics:

    root@beer:~/rev# file r1
	r1: data
	root@beer:~/rev# strings r1
	append
	print
	/tmp/tmpzyliz6qj
	<module>
So the file has `append`, `print`, and `<module>`. This reminds me a lot of Python, my initial guess was it's some sort of Python bytecode? A .pyc file maybe? However the signature was missing so I couldn't directly execute it as .pyc.
The alternative is to just run the file through a Python decompiler. There are a few available. I personally used an online one [here](https://python-decompiler.com/). It worked pretty well and I could copy the Python code to my interpreter and get the flag.

    In [1]: j = []
	   ...: j.append('C')
	   ...: j.append('T')
	   ...: j.append('F')
	   ...: j.append('{')
	   ...: j.append('d')
	   ...: j.append('3')
	   ...: j.append('f')
	   ...: j.append('3')
	   ...: j.append('d')
	   ...: j.append('d')
	   ...: j.append('b')
	   ...: j.append('1')
	   ...: j.append('5')
	   ...: j.append('8')
	   ...: j.append('d')
	   ...: j.append('c')
	   ...: j.append('4')
	   ...: j.append('b')
	   ...: j.append('b')
	   ...: j.append('4')
	   ...: j.append('3')
	   ...: j.append('b')
	   ...: j.append('e')
	   ...: j.append('7')
	   ...: j.append('0')
	   ...: j.append('8')
	   ...: j.append('c')
	   ...: j.append('3')
	   ...: j.append('3')
	   ...: j.append('a')
	   ...: j.append('9')
	   ...: j.append('d')
	   ...: j.append('4')
	   ...: j.append('f')
	   ...: j.append('6')
	   ...: j.append('5')
	   ...: j.append('}')

	In [2]: "".join(j)
	Out[2]: 'CTF{d3f3ddb158zx4bb43be708c33a9d4f65}'

