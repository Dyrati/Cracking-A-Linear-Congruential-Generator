## Cracking A Linear Congruential Generator
$$
r_{n+1} = \left(r_n*\texttt{0x41C64E6D + 0x3039}\right) \bmod 2^{32}
$$

Linear Congruential Generators (LCGs) are simple recurrence relations used to produce pseudo-random numbers.  Depending on the choice of parameters, the output can appear unpredictable and pass formal randomness tests.  However they are not cryptographically secure, as it is possible to directly determine the position of the generator based on its output.  This document explains how.
    
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

The first key observation is that the last bit of $r_{n+1}$ depends only on the last bit of $r_n$, because changing any of the other bits of $r_n$ changes $r_{n+1}$ by a multiple of 2.  By the same reasoning, the last two bits of $r_{n+1}$ depend only on the last two bits of $r_n$, the last three bits of $r_{n+1}$ depend only on the last 3 bits of $r_n$, and so on.

With our particular LCG, the last bit alternates between 0 and 1.  If instead it went from 0 to 0, or from 1 to 1, then it would get stuck on that value forever because the next value depends only on the previous value.  In an isolated system, a "cycle" is complete when it repeats its state, and its cycle length is the distance between repeats.

With any LCG mod $2^n$, whenever the last bit completes a cycle, the 2nd-to-last-bit may either be the same as it started, or different than it started.  If it's the same, then the cycle length of the last 2-bits would be equal to the cycle length of the last bit.  If it's different, its 2-bit cycle length is double its 1-bit cycle length.  This holds true for every n-bit cycle length (shown below).  

Let $L(n)$ = length of an n-bit cycle.  

1. Advancing $L(n)$ steps always leaves bit $0$ through bit $n-1$ unchanged.  

2. $L(n+1)$ cannot exceed $2L(n)$.  

3. $L(n+1)$ is always a multiple of $L(n)$, because the minimum distance between two identical values defines the cycle length.  

4. If $L(n+1) > L(n)$  

    a. $L(n+1) = 2L(n)$  
    
    b. Two points in the $L(n+1)$ cycle cannot map to the same point because that would mean the cycle length is less than $2L(n)$.  
    
    c. Two points in the $L(n+1)$ cycle that are a separated by $L(n)$ steps must differ in bit $n$ because the bits below that are the same.

Observing our particular LCG, we see that the 1-bit cycle is `0 -> 1`, the 2-bit cycle is `00 -> 01 -> 10 -> 11`, and the 3-bit cycle is `000 -> 001 -> 110 -> 111 -> 100 -> 101 -> 010 -> 011`.  Each cycle length so far has doubled, but that does not guarantee it will keep happening. It depends on the choice of multiplier and increment.

The second key observation is that it's possible to "skip ahead" an arbitrary number of advances by choosing different values for the LCG multiplier and increment.  To see this, imagine advancing just two steps forward:

$\text{m = multiplier, c = increment}$  
$r_{n+2} = (r_{n+1}m + c) \bmod 2^{32}$  
$r_{n+2} = (((r_{n}m + c) \bmod 2^{32})m + c) \bmod 2^{32}$  
$r_{n+2} = ((r_{n}m + c)m + c) \bmod 2^{32}$  
$r_{n+2} = (r_{n}m^2 + cm + c) \bmod 2^{32}$  
$r_{n+2} = (r_{n}(m^2 \bmod 2^{32}) + ((cm + c) \bmod 2^{32})) \bmod 2^{32}$  

With modular arithmetic, so long as each operation is integer multiplication or addition, you can apply the modulus operation anywhere you like without changing the result, provided you also apply the modulus operation at the end.  This is because every integer has the form `am + b`, and each multiplication/addition modulo `m` involving that integer is independent of `a`.

This leaves us with an equation for $r_{n+2}$ in terms of a new multiplier and increment:

$m_2 = m^2 \bmod 2^{32}$  
$c_2 = (cm + c) \bmod 2^{32}$  
$r_{n+2} = (r_nm_2 + c_2) \bmod 2^{32}$  

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

We can combine these arbitrarily to obtain $m_{x}$ and $c_{x}$ for any integer $x$.

$a \equiv b \mod m$ is shorthand for $a \bmod m = b \bmod m$.  
This is read as $a$ is **congruent to** $b$ modulo $m$.  

$r_{n+(a+b)} \equiv r_{n+a}m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} \equiv (r_nm_a  + c_a)m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} \equiv r_n(m_am_b) + (c_am_b+c_b) \mod 2^{32}$  

$m_{a+b} = (m_am_b) \bmod 2^{32}$  
$c_{a+b} = (c_am_b+c_b) \bmod 2^{32}$  

To obtain an arbitrary $r_n$, we can find the corresponding $m_n$ and $c_n$, and apply it to $r_0$.

$r_n = m_nr_0 + c_n$  
$r_n = c_n$

If bit $n$ of $r_{2^n}$ is 1, that means that advancing $2^n$ steps toggles bit $n$.  Advancing $2^n$ steps also completes an $n-1$ bit cycle, so if bit $n$ is toggled after those steps, that means that the $n+1$ bit cycle length is double the $n$ bit cycle length. So now we can quickly test whether our LCG has the maximum cycle length by confirming that bit $n$ of $r_{2^n}$ is always $1$.  After testing, it turns out that our LCG does have the maximum cycle length.

This means that it touches every number from $0$ to $2^{32}-1$ exactly once.  This also means that we can actually reverse the rng by using the formula for $r_{2^{32}-1}$.

We can also find the `r_count` given an arbitrary `r_value` using this method:
- start at bit position 0
- if the current bit is 1, advance `r_value` by $2^{position}$
- add 1 to the current position

When you're finished, `r_value` will be exactly $0$.  This works because each time you advance `r_value` by $2^{position}$, you toggle the current bit, and the bits below that are unchanged because you're advancing by a multiple of the cycle length of each of the lower bits.  You can then subtract the total advancements applied during that process from $2^{32}$ to find the original `r_count`.

---

### Python Implementation

```py
class LCG32:
    
    # Find m_n and c_n for each power of 2 up to 2^31
    def __init__(self, m, c):
        self.params = [(m, c)]
        for i in range(31):
            c = (c*m + c) & 0xFFFFFFFF
            m = m**2 & 0xFFFFFFFF
            self.params.append((m,c))
    
    # Find m and c for any number of advances, even negative
    def get_params(self, advances):
        advances &= 0xFFFFFFFF
        m_new = 1
        c_new = 0
        for i in range(advances.bit_length()):
            if advances & 1:
                m, c = self.params[i]
                m_new = (m_new*m) & 0xFFFFFFFF
                c_new = (c_new*m + c) & 0xFFFFFFFF
            advances >>= 1
        return m_new, c_new
    
    # Advance a value by an arbitrary count
    def advance(self, value, count):
        count &= 0xFFFFFFFF
        for i in range(count.bit_length()):
            if count & 1:
                m, c = self.params[i]
                value = (value*m + c) & 0xFFFFFFFF
            count >>= 1
        return value
    
    # Get value of an arbitrary count
    def value(self, count):
        return self.advance(0, count)
    
    # Get count of an arbitrary value
    def count(self, value):
        total = 0
        for i, (m, c) in enumerate(self.params):
            if not value: break
            if not value & (1 << i): continue
            value = (value*m + c) & 0xFFFFFFFF
            total += 1 << i
        return -total & 0xFFFFFFFF

rng = LCG32(0x41C64E6D, 0x3039)
```

(The parameters `0x41C64E6D` and `0x3039` are also used by glibc, and  `0x10DCD` and `0x1` are used by older versions of glibc.)
