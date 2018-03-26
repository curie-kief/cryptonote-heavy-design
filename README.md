## Resistance to FPGAs, weaknesses in cryptonight and general cryptonight-heavy design

**Please note. "Weakness" along with "break" are special terms in cryptography. They can vary in seriousness by 100's of orders of magnitude. Weakness described in this essay is not fatal. It is unlikely that someone will ever build a gigahash miner. Please return to your seats.**

## Design choices

### Why NOT to follow Monero's PoW?

**Monero's tweak**

Monero's programmers wrote their tweak as:

```
inline uint8_t test_a(uint8_t x)
{
    static const uint32_t table = 0x75310;
    const uint8_t index = (((x >> 3) & 6) | (x & 1)) << 1;
    return x ^ ((table >> index) & 0x3);
}
```

However, what they probably didn't realise is that it can be rewritten as:

```
inline uint8_t test_b(uint8_t x)
{
    uint8_t ret = x;
    ret ^= (((x ^ (x << 4)) & x) << 1) & 0x20;
    ret ^= (~((x | (x << 5)) ^ x) >> 1) & 0x10;
    return ret;
}
```

Which looks complicated, however it can in turn be written as:

```
bit[5] ^= (bit[6] NOR bit[1]) XOR bit[6]
bit[6] ^= (bit[5] XOR bit[1]) AND bit[5]
```

Which can then be implemented in hardware as:

<img src="http://pasteall.org/pic/show.php?id=8c9e9dd889fddb56a3cbdbe64b91871a">

Yes. You got it. Monero's tweak will run faster on hardware than on software.

**Cryptonite weakness**

Let us consider the transformation of the 2MiB buffer into a 200 byte output, later referred to as "implode". 
The final output, ignoring the ultimate 8 bytes, can be grouped into 12 16-byte groups, later referred to as "columns".

The first four columns are not transformed (columns 3 and 4 are used in implode as keys). This means that the variable output is constrained to 8 columns (5 to 12).

The weakness of implode is that it doesn't exhibit waterfall effect. Change in input in column n is only going to affect the output in column (n-1) % 8 + 5.

In light of this, we urgently call for more research into the "bypass" of cryptonite inner loop through statistical prediction, tabulation or differential cryptanalysis.

**Expert's opinion**

Monero project utterly ignored the opinions of FPGA programmers that poked their head into the discussion [[1]](https://github.com/monero-project/monero/pull/3253#issuecomment-367946170)[[2]](https://github.com/monero-project/monero/pull/3253#issuecomment-365418373). We thought that this was not a wise course of action.


### Design features and rationale

**Division half-step**

Our starting point was this opinion [[3]](https://github.com/sumoprojects/sumokoin/issues/91#issuecomment-373565085). 

We decided to call it a "half-step" since it is deliberately designed to be awkward in relation to the rest of the loop. Unlike the rest of the loop, which uses 16 byte wide reads and writes, it only uses a 12 byte wide read and 8 byte wide write. First 8 bytes, treated as a signed integer, become 'n' (numerator), the following 4 bytes, again treated as a signed integer become 'd' (denominator). Variable q (quotient) is calculated as `q = n / (d | 5)`. Value of 5 was chosen to avoid problematic cases like 0, 1, 2 and 4. 

8 bytes are written back to the scratchpad as `n ^= q`. While the address of the AES read is modified to be `d^q`. Note that the ax variable is *not* changed so it becomes uncoupled from its usual address function.
 
**Scratchpad increase to 4MB** 

This is the middle ground between single-thread performance (and therefore verification times), and FPGA capabilities [[4]](https://github.com/sumoprojects/sumokoin/issues/91#issuecomment-373586177). A welcome side effect of this change is reduction in mining performance (all-threads performance) without affecting verification times.

**Round decrease to 16384**

Again, keeping verification times at the same level was a major concern for us, hence we opted to decrease the number of rounds. The number of rounds is a fairly insignificant barrier to FPGAs. 

**Changes to implode and explode**

Implode was changed by addition of a shift_xor step. This step xors each column with the next one allowing the changes to propagate. The whole scratchpad is processed in two identical rounds to allow a single bit change, from the very end of the scratchpad, to propagate to all columns (waterfall effect).

Additionally, before "explode" and after "implode", columns 5 to 12 are encrypted and shift_xor'ed 16 times.

We don't consider the scratchpad changes to be an ultimate solution. They are intended as a stopgap measure while research on cryptonite cryptanalysis continues.
