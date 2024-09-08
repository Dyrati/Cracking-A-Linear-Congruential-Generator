## Cracking A Linear Congruential Generator
$$
r_{n+1} = \left(r_n*\texttt{0x41C64E6D + 0x3039}\right) \bmod 2^{32}
$$

The first key insight is that the last bit of $r_{n+1}$ depends only on the last bit of $r_n$, because changing any of the other bits of $r_n$ changes $r_{n+1}$ by a multiple of 2.  By the same reasoning, the last two bits of $r_{n+1}$ depend only on the last two bits of $r_n$, the last three bits of $r_{n+1}$ depend only on the last 3 bits of $r_n$, and so on.

With our particular LCG, the last bit alternates between 0 and 1.  If instead it went from 0 to 0, or from 1 to 1, then it would get stuck on that value forever because the next value depends only on the previous value.  In an isolated system, a "cycle" is complete when it repeats its state, and its cycle length is the distance between repeats.

With any LCG mod $2^n$, whenever the last bit completes a cycle, the 2nd-to-last-bit may either be the same as it started, or different than it started.  If it's the same as it started, its 2-bit cycle length would be equal to its 1-bit cycle length because the system repeated after a 1-bit cycle.  Otherwise, its 2-bit cycle length is double its 1-bit cycle length.  This holds true for every n-bit cycle length (shown below).  

Let $L(n)$ = length of an n-bit cycle.  

1. Advancing $L(n)$ steps always gives a number whose first $n$ bits are identical.  

2. $L(n)$ cannot exceed $2L(n-1)$.  

3. $L(n)$ is always a multiple of $L(n-1)$, because the minimum distance between two identical values defines the cycle length.  

4. If $L(n) > L(n-1)$  

    a. $L(n) = 2L(n-1)$  
    
    b. Two points cannot map to the same point, because that would create a cycle length less than $2L(n-1)$.  
    
    c. Two points in the n-bit cycle that are a distance of $L(n-1)$ apart must differ in the msb (most significant bit), because the remaining $n-1$ bits are the same

Observing our particular LCG, we see that the 1-bit cycle is `0 -> 1`, the 2-bit cycle is `00 -> 01 -> 10 -> 11`, and the 3-bit cycle is `000 -> 001 -> 110 -> 111 -> 100 -> 101 -> 010 -> 011`.  Each cycle length so far has doubled, but that does not guarantee it will keep happening. It depends on the choice of multiplier and increment.

The second key insight is that it's possible to "skip ahead" an arbitrary number of advances by choosing different values for the LCG multiplier and increment.  To see this, imagine advancing just two steps forward:

$\text{m = multiplier, c = increment}$  
$r_{n+2} = (r_{n+1}m + c) \bmod 2^{32}$  
$r_{n+2} = (((r_{n}m + c) \bmod 2^{32})m + c) \bmod 2^{32}$  
$r_{n+2} = ((r_{n}m + c)m + c) \bmod 2^{32}$  
$r_{n+2} = (r_{n}m^2 + cm + c) \bmod 2^{32}$  
$r_{n+2} = (r_{n}(m^2 \bmod 2^{32}) + ((cm + c) \bmod 2^{32})) \bmod 2^{32}$  

(With modular arithmetic, so long as each operation is integer multiplication or addition, you can apply the modulus operation anywhere you like without changing the result, provided you also apply the modulus operation at the end.  This is because every integer has the form `am + b`, and each multiplication/addition modulo `m` involving that integer is independent of `a`.)

This leaves us with an equation for $r_{n+2}$ in terms of a new multiplier and increment:

$m_2 = m^2 \bmod 2^{32}$  
$c_2 = (cm + c) \bmod 2^{32}$  
$r_{n+2} = (r_nm_2 + c_2) \bmod 2^{32}$  

Replacing $m$ and $c$ with $m_{2}$ and $c_{2}$ in the previous set of equations gives us an equation for $r_{n+4}$.  In this way, we can also obtain $r_{n+8}$, $r_{n+16}$, and so on.  We can also combine these arbitrarily to obtain an equation for $r_{n+x}$.

$a \equiv b \mod m$ is shorthand for $a \bmod m = b \bmod m$.  
This is read as $a$ is **congruent to** $b$ modulo $m$.  

$r_{n+(a+b)} \equiv r_{n+a}m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} \equiv (r_nm_a  + c_a)m_b + c_b \mod 2^{32}$  
$r_{n+(a+b)} \equiv r_n(m_am_b) + (c_am_b+c_b) \mod 2^{32}$  

$m_{a+b} = (m_am_b) \bmod 2^{32}$  
$c_{a+b} = (c_am_b+c_b) \bmod 2^{32}$  

$r_{n} \equiv r_0m_n + c_n \mod 2^{32}$  
$r_{n} = c_n$  

So now we can quickly test our LCG to see whether $L(n) = 2L(n-1)$ for n from $0$ to $31$ by observing $r_{2^n}$ (if the cycle did increase in size, then the $n$-th bit will be 1 by statement `4c`). It turns out that it does, meaning that it touches every number from $0$ to $2^{32}-1$ exactly once.  That means we can actually reverse the rng by using the formula for $r_{2^{32}-1}$.

We can also find the `r_count` given an arbitrary `r_value` using this method:
- start at bit position 0
- if the current bit is 1, advance `r_value` by $2^{position}$
- add 1 to current position

When you're finished, `r_value` will be exactly 0.  This works because each time you advance `r_value` by a power of 2, you're advancing by a multiple of the cycle length of each of the bits previously checked, meaning that they remain zero.  You can then subtract the total advancements applied during that process from $2^{32}$ to find the original `r_count`.

Summarizing everything:
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
    def params(self, advances):
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
        return advance(0, count)
    
    # Get count of an arbitrary value
    def count(self, value):
        total = 0
        for i, (m, c) in enumerate(self.params):
            if not value: break
            if not value & (1 << i): continue
            value = (value*m + c) & 0xFFFFFFFF
            total += 1 << i
        return -total & 0xFFFFFFFF

gs1_lcg = LCG32(0x41C64E6D, 0x3039)
gs2_lcg = LCG32(0x41C64E6D, 0x3039)
gs3_lcg = LCG32(0x10DCD, 0x1)
```

(The parameters `0x41C64E6D` and `0x3039` are also used by glibc, and  `0x10DCD` and `0x1` are used by older versions of glibc.)
