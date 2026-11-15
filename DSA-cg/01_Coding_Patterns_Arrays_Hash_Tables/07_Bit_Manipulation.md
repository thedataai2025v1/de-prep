# Bit Manipulation

**Pattern:** Bitwise operations  
**Use cases:** Flags, parity, masks, subsets, low-level representation, XOR cancellation.

## 1. Core operators

| Operator | Meaning |
|---|---|
| `&` | Bitwise AND |
| `|` | Bitwise OR |
| `^` | Bitwise XOR |
| `~` | Bitwise NOT |
| `<<` | Left shift |
| `>>` | Right shift |

For non-negative integers, `n & 1` tests the least significant bit. `n & (n - 1)` clears the lowest set bit.

## 2. Example: single number

Every value appears twice except one. Return the unique value.

### Brute force

Count frequencies with a dictionary: O(n) expected time and O(n) space.

### Optimized XOR

```python
def single_number(nums):
    result = 0
    for value in nums:
        result ^= value
    return result
```

Time O(n), auxiliary space O(1).

Why it works: `x ^ x = 0`, `x ^ 0 = x`, and XOR is associative and commutative.

## 3. Dry run

`[4, 1, 2, 1, 2]`:
- 0 XOR 4 = 4
- 4 XOR 1 = 5
- 5 XOR 2 = 7
- 7 XOR 1 = 6
- 6 XOR 2 = 4
Answer = 4.

## 4. Useful bit tricks

```python
def is_power_of_two(n):
    return n > 0 and (n & (n - 1)) == 0

def count_set_bits(n):
    if n < 0:
        raise ValueError("This implementation expects non-negative integers")
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count

def is_bit_set(n, bit):
    return (n & (1 << bit)) != 0

def set_bit(n, bit):
    return n | (1 << bit)

def clear_bit(n, bit):
    return n & ~(1 << bit)
```

Brian Kernighan's set-bit count runs O(number of set bits), bounded by O(number of bits).

## 5. Complexity and assumptions

For fixed-width machine integers, bitwise operations are typically treated as O(1). Python integers are arbitrary precision, so operations can scale with the number of machine words required for the integer.

## 6. Edge cases and mistakes

- Zero is not a power of two.
- Negative values and signed representation.
- Python `~n` is an unbounded signed complement; mask to a fixed width if required.
- Bit positions are usually zero-indexed; confirm convention.
- XOR uniqueness assumptions must be exact (e.g., every other number occurs exactly twice).

## 7. Interviewer–candidate discussion

**Interviewer:** Can XOR solve the problem if two numbers appear once and all others twice?

**Candidate:** Not directly with one accumulator; it returns the XOR of the two unique values. We can XOR all values, isolate a bit where the two unique values differ, then partition values by that bit and XOR each group.

## 8. Follow-ups

- Find two unique numbers when others appear twice.
- Missing number from range 0..n.
- Count set bits.
- Generate all subsets using bit masks.
- Check whether a bitmask contains a set of required flags.

## 9. Tests

```python
assert single_number([4, 1, 2, 1, 2]) == 4
assert single_number([7]) == 7
assert is_power_of_two(1) is True
assert is_power_of_two(16) is True
assert is_power_of_two(0) is False
assert is_power_of_two(18) is False
assert count_set_bits(0) == 0
assert count_set_bits(13) == 3
```

## Data Engineering connection

Bitmasks can encode compact feature flags or permissions. Define bit positions centrally, version the schema and avoid reusing a bit while old records may still contain it.
