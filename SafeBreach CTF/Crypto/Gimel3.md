
# Gimel3
The file is named c2.bmp. Inspecting it with file does not mention it being an actual bitmap image, just data:

    root@beer:~/crypto# file c3.bmp
	c3.bmp: data

After looking at the file with a hex editor. It's pretty clear to see the file magic was modified. The file starts with `@M` instead of `BM`. After changing the first character to `B` you can view the bitmap.

![The bitmap](https://i.imgur.com/ukk5NUo.png)
The image does not seem interesting. Most steg analysis tools did not find anything interesting. My experience tells that if the data is not easily found in the image using strings/stegdetect, it's usually data in the LSBs.

Hiding data in LSBs of images is pretty straightforward, the "secret" (or in this case, flag) is embedded into the least significant bits in the image. If you rebuild the sequence of those LSBs into bytes again, you can find the secret. Here's my Python script:

    data = open('c3.bmp', 'rb').read()
	# Skip BMP header
	data = data[14:]

	bits = ""
	# Get all Least Signifcant Bits
	for byte in data:
	    bits += str(byte & 0x1)

	# Convert to bytes
	byte_string = ""
	for i in range(0, len(bits), 8):
	    byte = int(bits[i:i+8], 2)
	    byte_string += chr(byte)

	# Find the flag!
	index = byte_string.find("CTF")
	print(byte_string[index:index+37]) # 37 is fixed flag length

Execute with Python3:

    root@beer:~/crypto# python3 lsb.py
	CTF{a391ec614b722c5abdbbf94e006a715c}
