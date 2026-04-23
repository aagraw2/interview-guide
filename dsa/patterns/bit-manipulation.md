# Bit Manipulation Patterns

Bit operations enable O(1) tricks and efficient counting. Master these fundamental operations.

---

## Bit Basics

```
AND (&):  1 & 1 = 1, else 0     (both must be 1)
OR  (|):  0 | 0 = 0, else 1     (at least one 1)
XOR (^):  same = 0, diff = 1    (exactly one 1)
NOT (~):  flip all bits
LEFT SHIFT (<<):  multiply by 2
RIGHT SHIFT (>>): divide by 2

Key properties:
a ^ 0 = a           (XOR with 0 keeps value)
a ^ a = 0           (XOR with self gives 0)
a & (a-1)           (clears lowest set bit)
a & (-a)            (isolates lowest set bit)
```

---

## Pattern 1: XOR for Single Element

### Core Idea
XOR cancels pairs: a ^ a = 0. Use to find unpaired elements.

### When to Use
- Find single number among pairs
- Find missing number
- Two elements appear once

### Mental Model
```
XOR all elements
pairs cancel out → unpaired remains
```

### Example: Single Number
```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num;
    }
    return result;
}
```

### Example: Missing Number
```java
public int missingNumber(int[] nums) {
    int result = nums.length;
    for (int i = 0; i < nums.length; i++) {
        result ^= i ^ nums[i];
    }
    return result;
}
```

### Example: Single Number III (Two Single Numbers)
```java
public int[] singleNumber(int[] nums) {
    int xor = 0;
    for (int num : nums) {
        xor ^= num;
    }

    // Find rightmost set bit (differs between two numbers)
    int rightmostBit = xor & (-xor);

    int a = 0, b = 0;
    for (int num : nums) {
        if ((num & rightmostBit) == 0) {
            a ^= num;
        } else {
            b ^= num;
        }
    }

    return new int[] {a, b};
}
```

---

## Pattern 2: Count Set Bits (Hamming Weight)

### Core Idea
Count 1s in binary representation.

### When to Use
- Number of 1 bits
- Hamming distance
- Counting bits for range

### Method 1: Check each bit
```java
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        count += n & 1;
        n >>>= 1;  // Unsigned right shift
    }
    return count;
}
```

### Method 2: Clear lowest set bit (faster)
```java
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1);  // Clear lowest set bit
        count++;
    }
    return count;
}
```

### Example: Hamming Distance
```java
public int hammingDistance(int x, int y) {
    int xor = x ^ y;  // Different bits are 1
    int count = 0;
    while (xor != 0) {
        xor &= (xor - 1);
        count++;
    }
    return count;
}
```

---

## Pattern 3: Counting Bits (DP with Bits)

### Core Idea
dp[i] = dp[i >> 1] + (i & 1). Each number's bit count relates to its right-shifted value.

### When to Use
- Count bits for 0 to n
- Build on previous results

### Example: Counting Bits
```java
public int[] countBits(int n) {
    int[] dp = new int[n + 1];

    for (int i = 1; i <= n; i++) {
        dp[i] = dp[i >> 1] + (i & 1);
        // OR: dp[i] = dp[i & (i-1)] + 1
    }

    return dp;
}
```

---

## Pattern 4: Power of Two/Four Check

### Core Idea
Power of 2 has exactly one set bit. n & (n-1) == 0.

### When to Use
- Check if power of 2/4/8
- Validate bit patterns

### Example: Power of Two
```java
public boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

### Example: Power of Four
```java
public boolean isPowerOfFour(int n) {
    // Power of 2 AND set bit at odd position (0x55555555 = 0101...0101)
    return n > 0 && (n & (n - 1)) == 0 && (n & 0x55555555) != 0;
}
```

---

## Pattern 5: Bit Manipulation for Addition

### Core Idea
XOR for sum without carry. AND + left shift for carry.

### When to Use
- Add without + operator
- Implement arithmetic

### Example: Sum of Two Integers
```java
public int getSum(int a, int b) {
    while (b != 0) {
        int carry = (a & b) << 1;
        a = a ^ b;  // Sum without carry
        b = carry;
    }
    return a;
}
```

---

## Pattern 6: Reverse Bits

### Core Idea
Extract bits from right, place in reversed position.

### When to Use
- Reverse bit order
- Mirror binary representation

### Example: Reverse Bits
```java
public int reverseBits(int n) {
    int result = 0;
    for (int i = 0; i < 32; i++) {
        result <<= 1;
        result |= (n & 1);
        n >>>= 1;
    }
    return result;
}
```

---

## Pattern 7: Get/Set/Clear Bit

### Core Idea
Use masks to manipulate specific bits.

### When to Use
- Modify specific bits
- Check specific bits

### Template
```java
// Get bit at position i
boolean getBit(int n, int i) {
    return (n & (1 << i)) != 0;
}

// Set bit at position i to 1
int setBit(int n, int i) {
    return n | (1 << i);
}

// Clear bit at position i
int clearBit(int n, int i) {
    return n & ~(1 << i);
}

// Toggle bit at position i
int toggleBit(int n, int i) {
    return n ^ (1 << i);
}
```

---

## Pattern 8: Bitmask for Subsets

### Core Idea
Use bitmask to represent subset. Bit i = 1 means include element i.

### When to Use
- Generate all subsets
- Subset enumeration
- State compression DP

### Example: Generate All Subsets
```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    int n = nums.length;

    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) {
                subset.add(nums[i]);
            }
        }
        result.add(subset);
    }

    return result;
}
```

---

## Pattern 9: Maximum XOR

### Core Idea
Build trie of binary representations. Greedily choose opposite bit.

### When to Use
- Maximum XOR of two numbers
- XOR optimization

### Example: Maximum XOR of Two Numbers
```java
public int findMaximumXOR(int[] nums) {
    int maxXor = 0;
    int mask = 0;

    for (int i = 31; i >= 0; i--) {
        mask |= (1 << i);
        Set<Integer> prefixes = new HashSet<>();

        for (int num : nums) {
            prefixes.add(num & mask);
        }

        int candidate = maxXor | (1 << i);
        for (int prefix : prefixes) {
            if (prefixes.contains(prefix ^ candidate)) {
                maxXor = candidate;
                break;
            }
        }
    }

    return maxXor;
}
```

---

## Quick Reference

### Common Operations
| Operation | Code | Description |
|-----------|------|-------------|
| Check bit | `n & (1 << i)` | Is bit i set? |
| Set bit | `n \| (1 << i)` | Set bit i to 1 |
| Clear bit | `n & ~(1 << i)` | Set bit i to 0 |
| Toggle bit | `n ^ (1 << i)` | Flip bit i |
| Clear lowest set | `n & (n-1)` | Remove rightmost 1 |
| Isolate lowest set | `n & (-n)` | Get rightmost 1 |
| Check power of 2 | `n & (n-1) == 0` | Has single 1 bit |

### Useful Constants
```java
0x55555555 = 01010101...  // Odd positions
0xAAAAAAAA = 10101010...  // Even positions
0x33333333 = 00110011...  // Pairs
0x0F0F0F0F = 00001111...  // Nibbles
```

### Signed vs Unsigned
```java
>>   // Signed right shift (preserves sign)
>>>  // Unsigned right shift (fills with 0)
```

## When to Use Bit Manipulation
- Find single/missing element in pairs
- Power of 2 operations
- Set/subset representation
- Low-level optimization
- Replacing arithmetic with bitwise
