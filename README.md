## Cracking A Linear Congruential Generator

$$r_0 = 0$$
$$r_{n+1} = \left(r_{n}m + c\right) \bmod 2^{32}$$

Linear Congruential Generators (LCGs) are simple recurrence relations used to produce pseudo-random numbers.  Depending on the choice of parameters $m$ (multiplier) and $c$ (increment), the output can appear unpredictable and pass formal randomness tests.  However they are not cryptographically secure, as it is possible to directly determine the position of the generator based on its output.  This document explains how.

$$m = \texttt{0x41C64E6D}$$  
$$c = \texttt{0x3039}$$

|$r_i$|dec|hex|bin|
|-|-|-|-|
|$r_0$|0|00000000|00000000000000000000000000000000|
|$r_1$|12345|00003039|00000000000000000011000000111001|
|$r_2$|3554416254|D3DC167E|11010011110111000001011001111110|
|$r_3$|2802067423|A70427DF|10100111000001000010011111011111|
|$r_4$|3596950572|D6651C2C|11010110011001010001110000101100|
|$r_5$|229283573|0DAA96F5|00001101101010101001011011110101|
|$r_6$|3256818826|C21F1C8A|11000010000111110001110010001010|
|$r_7$|1051550459|3EAD62FB|00111110101011010110001011111011|
|$r_8$|3441282840|CD1DCF18|11001101000111011100111100011000|

The first key observation is that the last bit of $r_{n+1}$ depends only on the last bit of $r_n$, because changing $r_n$ by a multiple of 2 also changes $r_{n+1}$ by a multiple of 2.  Similarly, the last two bits of $r_{n+1}$ depend only on the last two bits of $r_n$, the last three bits of $r_{n+1}$ depend only on the last 3 bits of $r_n$, and so on.  Essentially, the lower $b$ bits of $r_{n+1}$ are unaffected by any bits above that in $r_n$.  The lower $b$ bits of the output can be viewed in isolation.

With our choice of parameters, the last bit alternates between 0 and 1.  If instead it went from 0 to 0, or from 1 to 1, then it would get stuck on that value forever because the next value of that bit depends only on the previous value of that bit.

A "cycle" is complete when something repeats its state, and its cycle length is the distance between repeats.  With any LCG mod $2^n$, whenever the last bit completes a cycle, the 2nd-to-last-bit may either be the same as it started, or different than it started.  If it's the same, then the cycle length of the last 2-bits would be equal to the cycle length of the last bit.  If it's different, its 2-bit cycle length is double its 1-bit cycle length.  This holds true for every n-bit cycle length (shown below).  

Let $L(n)$ = length of an n-bit cycle.  

1. Advancing $L(n)$ steps always leaves bit $0$ through bit $n-1$ unchanged (by definition).  (*NOTE: Bit 0 is the 1's place bit.*)

2. $L(n+1)$ cannot exceed $2L(n)$ because it only adds one bit, which can no more than double the number of possible states.  

3. $L(n+1)$ is always a multiple of $L(n)$, because traversing an $L(n+1)$ cycle means all $n+1$ bits returned to their original states, which includes the lower $n$ bits.

4. If $L(n+1) > L(n)$  

    a. $L(n+1) = 2L(n)$ by statements 2 and 3
    
    b. Two points in the $L(n+1)$ cycle cannot map to the same point, because there would be no way to return to *both* initial points from the mapped point.
    
    c. Two points in the $L(n+1)$ cycle that are a separated by $L(n)$ steps must differ in bit $n$, because otherwise, those two points would be the same point, which would mean $L(n+1) = L(n)$.

Observing our output table, we see that the 1-bit cycle is `0 -> 1`, the 2-bit cycle is `00 -> 01 -> 10 -> 11`, and the 3-bit cycle is `000 -> 001 -> 110 -> 111 -> 100 -> 101 -> 010 -> 011`.  Each cycle length so far has doubled, but that does not guarantee it will keep happening. It depends on the choice of multiplier and increment.

The second key observation is that it's possible to "skip ahead" an arbitrary number of advances by choosing different values for the multiplier and increment.  To see this, imagine advancing just two steps forward:

$r_{n+2} = r_{n+1}m + c \mod 2^{32}$  
$r_{n+2} = (r_{n}m + c)m + c \mod 2^{32}$  
$r_{n+2} = r_{n}m^2 + cm + c \mod 2^{32}$  

$m_2 = m^2 \mod 2^{32}$  
$c_2 = cm + c \mod 2^{32}$  
$r_{n+2} = r_{n}m_2 + c_2 \mod 2^{32}$  

This leaves us with an equation for $r_{n+2}$ in terms of a new multiplier and increment.
Replacing $m$ and $c$ with $m_{2}$ and $c_{2}$ in the previous equations gives us $m_4$ and $c_4$, and we can continue doubling up to $m_{2^{31}}$ and $c_{2^{31}}$.  

|i|$m_i$|$c_i$|
|-|-|-|
|$2^{0}$|41C64E6D|00003039|
|$2^{1}$|C2A29A69|D3DC167E|
|$2^{2}$|EE067F11|D6651C2C|
|$2^{3}$|CFDDDF21|CD1DCF18|
|$2^{4}$|5F748241|65136930|
|$2^{5}$|8B2E1481|642B7E60|
|$2^{6}$|76006901|1935ACC0|
|$2^{7}$|1711D201|B6461980|
|$2^{8}$|BE67A401|1EF73300|

We can also combine any two pairs of multipliers and increments: 

$r_{n+(a+b)} = r_{n+a}m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} = (r_nm_a  + c_a)m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} = r_n(m_am_b) + (c_am_b+c_b) \mod 2^{32}$  

$m_{a+b} = m_am_b \mod 2^{32}$  
$c_{a+b} = c_am_b+c_b \mod 2^{32}$  

Applying this formula repeatedly using $m_i$ and $c_i$ from our powers of 2 table allows us to obtain $m_{x}$ and $c_{x}$ for any whole number $x$.  To obtain an arbitrary $r_x$, we can find the corresponding $m_x$ and $c_x$, and apply it to $r_0$.

$r_x = r_{0}m_x + c_x \mod 2^{32}$  
$r_x = c_x$

Advancing $2^n$ steps always completes an $n$ bit cycle.  If bit $n$ of $r_{2^n}$ is 1, that means that advancing $2^n$ steps toggles bit $n$, which means that $L(n+1) = 2L(n)$.  We can test whether the cycle length doubles every time by checking whether bit $n$ of $r_{2^n}$ is 1 for each $n$ from 0 to 31. Our choice of multiplier and increment does pass this test, which means it also has the maximum cycle length.

This means that it touches every number from $0$ to $2^{32}-1$ exactly once.  This also means that we can reverse the rng by using the formula for $r_{2^{32}-1}$.

We can also find the `r_count` given an arbitrary `r_value` using this method:
- start at bit position 0 (1's place)
- if the bit at the current position is 1, advance `r_value` by $2^{position}$
- add 1 to the position

At the end of this process, `r_value` will be exactly $0$.  This works because each time `r_value` is advanced by $2^{position}$, the bit at that position is toggled, and the bits below that position are unchanged because the advancement is a multiple of the cycle length of each of the lower bits.  The total advancements equals the distance to $r_0$, which you can then subtract from $2^{32}$ to find the original `r_count`.

---

### Python Implementation

```py
class LCG32:
    
    # Find m_n and c_n for each power of 2
    def __init__(self, m, c):
        bit_length = 32
        self.bitmask = 2**bit_length - 1
        self.params = [(m, c)]
        for i in range(bit_length - 1):
            c = (c*m + c) & self.bitmask
            m = m**2 & self.bitmask
            self.params.append((m,c))
    
    # Find m_x and c_x for any integer x
    def get_params(self, x):
        x &= self.bitmask
        m_x = 1
        c_x = 0
        for i in range(x.bit_length()):
            if x & 1:
                m, c = self.params[i]
                m_x = (m_x*m) & self.bitmask
                c_x = (c_x*m + c) & self.bitmask
            x >>= 1
        return m_x, c_x
    
    # Get value of an arbitrary count
    def value(self, count, init=0):
        count &= self.bitmask
        for i in range(count.bit_length()):
            if count & 1:
                m, c = self.params[i]
                init = (init*m + c) & self.bitmask
            count >>= 1
        return init
    
    # Get count of an arbitrary value
    def count(self, value):
        advances = 0
        bitmask = 1
        for m, c in self.params:
            if not value: break
            if value & bitmask:
                value = (value*m + c) & self.bitmask
                advances += bitmask
            bitmask <<= 1
        return -advances & self.bitmask

rng = LCG32(0x41C64E6D, 0x3039)
print(rng.value(3)) # prints 2802067423
print(rng.count(2802067423)) # prints 3
```

(The parameters `0x41C64E6D` and `0x3039` are also used by glibc, and  `0x10DCD` and `0x1` are used by older versions of glibc.)
