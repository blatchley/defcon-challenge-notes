# defcon-challenge-notes
Notes on crypto defcon challenges, playing under the Norsecode tag as a member of Kalmarunionen.
These challenges were solved together with several other Norsecode members.

# Nooombers
This challenge took us a lot longer than it should have. We quickly identified that there were 10 commands we could work with, and that the 144 char sequences each denoted some kind of "number" or group element.

To begin with we started by experimenting with the operations given. We found that `0` generated one of 7 unique random numbers.
We then started playing with chaining functions, trying to establish whether any familiar group laws seemed to apply.

We found that `1(1(x)) == x` and `2(2(x)) == x`, that is 1 and 2 are their own inverses. We also found that `3(x,y) = 3(y,x)` and `4(x,y) = 4(y,x)`. 

After playing around with these operations for a while, we deduced that `1(x)` appeared to create the inverse of `x` with respect to operation `3`, and likewise `2` created an inverse with respect to operation `4`. At this point we assumed that we were in some kind of field, where 3 and 4 were the group law operations, and 1 and 2 were the inverse operations.

We also note that calling any of the functions has them return an extra variable, which is always fixed between operations 1-4, so we assume that this is a modulus of some kind. We confirm this by showing that calling 3 or 4 with two of this value returns a value which behaves like a `0` element. (Identity on 3, and absorbing on 4,) and we can also create a multiplicative identity by calling `4(x, 2(x))`.

We also note that 5 and 6 appear to be doing some operations with a different modulus. After some experimentation, we deduced that 5 behaved like 4, so was modular multiplication, and 6 did something different. Experimenting with our additive and multiplicative identities we deduced that 6 was modular exponentiation. We then also showed that the larger modulus was a multiple of the lower one + 1, and we assume that they simple represent primes of the form p = a*q + 1, where p and q are primes.

We then spent some time implementing a binary search to try and leak the moduli, but it turned out that they changed every run so this was a waste.


At this point we realised that this was all kind of demonstrated in interaction 1, which we hadn't looked at properly until now. psyduck.


From here we used the trace of operation 8 and 9 to see what they did. 

We found that they had some hard coded constants, and performed the following operation

given an S, s and r, `r == g^(S * s^-1 mod q) * y^(r * s^-1 mod q) mod p`, with operation 9 being the same except with S fixed to some arbitrary value.

This looks a lot like DSA signatures, so we assume 8 checks any signature, and 9 checks the one we have to forge.
We then guessed 7 does signing, and this checked out, as signing something arbitrary with it satisfied 8, and signing the value S (leaked via calling 9,) gave access denied. 

Final thing we noticed was that the `k` value presumably used in 7 would be called by `0`, but  `0` only gave one of 7 values, so we just guess a value for k and assume it's correct, and build a signature on S with it, and keep pinging remote til the guessed value matches `k`. (1/7 chance.)

Finally this needed to be done in very few requests, as 9 gets disabled after ~12 commands. our final script was something like 
```
0()                → k
2(k)               → k^-1
9(k, k)            → S
7(k)               → r, 1 + k^-1xr
4(k, k^-1)         → 1
1(1)               → -1 
3(1 + k^-1xr, -1)  → k^-1xr
4(k^-1xr, k)       → xr
3(xr, S)           → S + xr
4(S + xr, k^-1)    → k^-1(S + xr)
9(r, k^-1(S + xr)) → flag
```


# qoo or ooo

We solved the first one by just randomly selecting quantum coin, and flipping left if opponent was 0, and right if opponent was 1. This got lucky and won, and from transcript we could see key was only 12 bits, as the shared key only has a bit added when the opponents bet is 1 or something.

Brute all possible AES keys and win.

# Qoo or ooo 2
I didn't help solve this one, but it looks like solution iterated strategies to find a good one, and found that winning 4 times in a row with a 0 bet, then always picking quantum coin and not flipping, had a decent winrate, and got through to chat room quite fast. This then showed the key with some smart trick i missed, and got flag.

# Smart Crypto
TLDR because I ran out of time;

We basically created a pytorch version of the network, and downloaded the webpage to use as a plaintext. Just running our network (which both build the model and also learned keys which decrypted the plaintext.) This explicitly ignored the "key blocks", dropping them from the inputs.

We noticed that this worked great until chunk 103/block 1632~ at which point it broke. We assumed they inserted flag here. 

(3 hours later)

We deduced flag length was 260 bytes, and after that we're learning everything after the flag perfectly as well.

(3 hours later)

we worked out that we can't brute the key by itself, and pivoted. Instead we used the keys which we decrypted from the other blocks, and instead saved all the keys we decrypted, then started again with a new model which was training to use the provided keys, and the provided cipher/plaintext, and tried to find a model which could decrypt using those keys.

we then used this model and the key which our first model found at the end of block 102 to decrypt block 103, we added the decryption of that (with the typos fixed,) to the plaintext, and reran first model to get the key at the end of 103, then used that key in the second model to decode second half which had the flag.
