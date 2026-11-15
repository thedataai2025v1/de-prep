# Two-Pass Technique

**Pattern:** Two Pass / Precompute then consume  
**Use cases:** Prefix/suffix products, validation then construction, frequency then output, left/right contributions.

## 1. Concept

A two-pass algorithm scans the input twice, often because the second pass needs information that is only available after examining the whole input. It can replace nested scans while keeping logic simple.

Two passes are not automatically slower than one pass in a meaningful way. O(2n) simplifies to O(n); clarity and state requirements matter.

## 2. Example: Product of Array Except Self

For each index, return the product of all values except the value at that index. Do not use division unless the interviewer explicitly allows it.

### Brute force

```python
def product_except_self_bruteforce(nums):
    result = []
    for i in range(len(nums)):
        product = 1
        for j in range(len(nums)):
            if i != j:
                product *= nums[j]
        result.append(product)
    return result
```

Time O(n²), output O(n).

### Optimized prefix then suffix

```python
def product_except_self(nums):
    n = len(nums)
    result = [1] * n

    prefix = 1
    for i in range(n):
        result[i] = prefix
        prefix *= nums[i]

    suffix = 1
    for i in range(n - 1, -1, -1):
        result[i] *= suffix
        suffix *= nums[i]

    return result
```

Time O(n), auxiliary space O(1) excluding the output array.

## 3. Dry run

Input `[1, 2, 3, 4]`:
- Prefix pass writes `[1, 1, 2, 6]`.
- Suffix pass from right: index 3 stays 6; index 2 becomes 8; index 1 becomes 12; index 0 becomes 24.
- Result `[24, 12, 8, 6]`.

## 4. Correctness argument

At index i, the first pass stores the product of all values strictly left of i. The reverse pass multiplies by the product of all values strictly right of i. Their product is exactly the product of all elements except `nums[i]`.

## 5. Complexity

- Brute force: O(n²) time.
- Optimized: O(n) time.
- Output: O(n); extra working state O(1).

## 6. Edge cases and mistakes

- One element → `[1]`.
- Zeros: algorithm naturally handles one or multiple zeros.
- Negative values.
- Empty input.
- Confusing output space with auxiliary space.
- Accidentally including the current element in its prefix or suffix.

## 7. Interviewer–candidate discussion

**Interviewer:** Why not divide the total product by each value?

**Candidate:** Division fails when zeros are present and may be disallowed. Prefix and suffix products handle zeros without special-case division and use linear time.

## 8. Follow-ups

- Sum of elements except self.
- Left and right maximum arrays.
- Trapping Rain Water using prefix/suffix maxima.
- For each item, count smaller elements on left and right.
- Explain when a one-pass algorithm is possible and what state it would need.

## 9. Tests

```python
assert product_except_self([1, 2, 3, 4]) == [24, 12, 8, 6]
assert product_except_self([-1, 1, 0, -3, 3]) == [0, 0, 9, 0, 0]
assert product_except_self([2]) == [1]
assert product_except_self([0, 0]) == [0, 0]
assert product_except_self([]) == []
```
