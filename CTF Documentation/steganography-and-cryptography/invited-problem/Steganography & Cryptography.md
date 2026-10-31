**Steganography \& Cryptography Solution**

**---------------------------------------------------------------------------------------------------------------------------------------------------**
**Step 1:**

Think through to identify the image of interest that might have some information enbeded on it. You will find the I&M Logo.png being of interest. It is the unique one as others are .jpeg and the other .exe (disruptor)


**Step 2:**

Use any tool eg. https://stylesuxx.github.io/steganography/ to decode the message hidden in the image, you will find the text: YzNsdWRIdDJZWHBmYkdKb0xYUmlaeTEyWnkxd1ltRjBaWEpuYUhsdVozWmlZV1o5



**Step 3:**

Notice it looks like Base64. Decode, you will find the text:

c3ludHt2YXpfbGJoLXRiZy12Zy1wYmF0ZXJnaHluZ3ZiYWZ9



**Step 4:**

Looks like Base64 again. Decode again, you will find the text:

synt{vaz\_lbh-tbg-vg-pbaterghyngvbaf}



**Step 5:**

Apply ROT13 using clue from hint.

synt{vaz\_lbh-tbg-vg-pbaterghyngvbaf}

↓

flag{inm\_you-got-it-congratulations}



