OpenTheFlag WriteUp

XxcoralloxX

## Step 1: Analyze the binary

Here we go:

With checksec on peda we can see what kind of protection are on:\
![AltText](https://i.gyazo.com/714cd40ebff5d544322221ac06f71707.png)\
NX is on, so probably this won't allow us to do shellcoding.

Let's see what we have:\
![AltText](https://i.gyazo.com/205f20c3df6300e9269b0727445d7d34.png)\
Few functions, good, let's look them one by one

main:

![AltText](https://i.gyazo.com/2a037008d9eb4d64f77c88b9fd5b6599.png)

Nothing in particular to notice, so we should move our attention on the function pwnme

pwnme:

![AltText](https://i.gyazo.com/2e8bf518da3e1c08d84ccda8da90d502.png)

A fgets is here, probably this the vulnerability.

iShouldNotBeHere:

![AltText](https://i.gyazo.com/6d394616a3fecd742f950a56a4d940be.png)

This is also interesting, as we can see it:
1) Print stuff 
2) check if the len of a string passed as argumetn is > 4 otherwise it exit
3) Open  file
4) read from the file
5) print the content of the file.

![AltText](https://i.gyazo.com/9033f77e80aeec5447f535298a3894b6.png)

So this is probably our way to win, we just need to give to that function in a string the path of the flag.

Last thing to check

stuff:

![AltText](https://i.gyazo.com/fa11cafb82928a534247374dbb78b3d8.png)

This function has no sense, but we can notice that there are 2 gadgets.

So, what we got from our analysis?
1) We have a possible buffer overflow in pwnme
2) We have a function that wants a string as parameter, and it print the content of a file with the name of that string
3) We have some gadgets


## Step 2: Think how to expoloit 

The hint told us that the flag is in "/etc/flag".

The main problem here is "How can we produce that string?"

First, let's check if it is already there:

![AltText](https://i.gyazo.com/40a1a6b4c422283273bcf7a3f18cc67e.png)

Sad, so we need to craft it.

NOTE: You **CAN NOT** simply write it into the stack since the stack address **CHANGE** every execution.

Here is where our gadgets come handy:

![AltText](https://i.gyazo.com/fa11cafb82928a534247374dbb78b3d8.png)

The red one, move into edi and ecx two elements of the stack.

The yellow one, mov ebx into the memory address of edi.

So we need something more, we can't just use those 2 gadgets, to write what we want. (since we can't control ebx right now).


We need to do a deeper search to see if we find how to fix this problem

With ROPgadget, we can see all the possible gadgets, and we can filter them with a grep:

![AltText](https://i.gyazo.com/8bf6e2b70c0f48b401ff4d88814d94ed.png)

Here we go. 

A nice mov ebx, ecx

If we put together our gadgets we get:

```
pop edi
pop ecx
mov ebx, ecx
mov [edi], ebx
```

Moving in edi the address where we want to write, and in ebx what to write, we can basically write everywhere (more or less).

We are almost done.

Last question: "Where do I write it?"

We need a space of 10 chars to write "/etc/flag" (including /0).

So:

![AltText](https://i.gyazo.com/fcf218269f752172bb71ffd680711924.png)

We, of course, need a readable and writeable area and as we can see, memory has that permission between 0x0804bf00 and 0x0804c048.

We want to choose a segment that causes less damage possible, .data for example, it is big enough to store our string.

Notice that even if we could write everywhere in that memory address, putting our string in the .got, for example, could broke every function calls.

Good, the plan is done.

We know what to do, what to write, where to write, and how to write it.

Last info to retrive:

![AltText](https://i.gyazo.com/83050768edd4bc60b7ff55d249da7ca3.png)

## Last step:
Let's make our exploit:

The payload will be:

```
PADDING = "A"*44;
pop = address of the pop pop gadget we found
area = .data address
mov1 = addres of mov ebx, ecx gadget
mov2 = addres of mov [edi], ebx gadget
win = addres of the iShouldNotBeHere

PADDING + pop + area + "/etc" + mov1 + mov2 + pop + (area+4) + "/fla" + mov1 + mov2 + pop + (area+8) + "g\x00\x00\x00" + mov1 +mov2 + win + "AAAA" + area  
```


If you understood everything probably will be useless to read from now on, otherwise, if you want to understand better how the rop chain work, try to read the following part:

## How the exploit work:

We prepare the stack as follow:

![AltText](https://i.gyazo.com/77ea80446d763dad4bc6a43741325208.png)

---

The RET of the pwnme function will take the first address from the stack, and will continue the execution from that point.\

![AltText](https://i.gyazo.com/d92b4fabb991f46d1461b0e31214880c.png)\
The first gadget will move into the two registers the area_addr and the "/etc" string.\
then the return, will take again the first address from the stack and will continue the execution. \
NOTE: The stack, lost its first 3 elements because of the ret and the 2 pop.\

---
The first mov will put in ebx the content of ecx

![AltText](https://i.gyazo.com/81648db73a54ad6c873987ab1eec018a.png)

---
The second mov will put "/etc" in the area_addr

![AltText](https://i.gyazo.com/fa2a3f5c0ea8030a20eabcd47903378a.png)

Then, the ret will take out the pop gadget, and the same will happen, until the string
"/etc/flag" is completely written.

---
At this point in the stack there will be only 

![AltText](https://i.gyazo.com/56efcea57c8bc6fc09b12dcb3482f6db.png)\
Win_function (iShouldNotBeHere) will be executed, with "AAAA" as return addres and area_addr as argument.\
iShouldNotBeHere open the file, read the flag, and write it as output.\

Hope this will be helpful. \
Good luck!
