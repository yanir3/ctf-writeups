
# Gimel2
You are given a file. After inspecting it, it seems like raw data, 37 bytes long, just like the last flag.
The length is a strong hint that this file is just XORed with some key. All you need to do is brute-force it.
I crafted a small script to do that:

    data = open('c2', 'rb').read()
    
    for xor_key in range(256):
        result = "".join([chr(char ^ xor_key) for char in data])
        if "CTF" in result:
            print(result)

Execute it:

        root@beer:~/crypto# python3 xor.py
        CTF{cdd4259ef808e4d4f5dce4678abd26e1}
