# Arrays & Hashing Patterns

Arrays and HashMaps are the foundation of most DSA problems. Master these patterns first.

---

## Pattern 1: HashMap for O(1) Lookup

### Core Idea
Use a HashMap to store values/indices for constant-time lookups instead of nested loops.

### When to Use
- Need to find complement/pair that satisfies a condition
- "Find two elements that sum to X"
- Need to check existence quickly
- Reduce O(n²) brute force to O(n)

### Mental Model
```
for each element:
    check if complement exists in map
    if yes → found answer
    if no → store current element in map
```

### Generic Template
```java
public int[] findPair(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];

        if (map.containsKey(complement)) {
            return new int[] { map.get(complement), i };
        }

        map.put(nums[i], i);
    }

    return new int[] {};
}
```

### Example: Two Sum
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];

        if (map.containsKey(complement)) {
            return new int[] { map.get(complement), i };
        }

        map.put(nums[i], i);
    }

    return new int[] {};
}
```

---

## Pattern 2: HashSet for Duplicate Detection

### Core Idea
Use a HashSet for O(1) membership checking to detect duplicates or unique elements.

### When to Use
- Check if duplicates exist
- Find unique elements
- Check membership in a collection

### Mental Model
```
for each element:
    if element in set → duplicate found
    else → add to set
```

### Generic Template
```java
public boolean hasDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();

    for (int num : nums) {
        if (seen.contains(num)) {
            return true;
        }
        seen.add(num);
    }

    return false;
}
```

### Example: Contains Duplicate
```java
public boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();

    for (int num : nums) {
        if (!seen.add(num)) {  // add() returns false if already exists
            return true;
        }
    }

    return false;
}
```

---

## Pattern 3: Frequency Count

### Core Idea
Count occurrences of elements using a HashMap or array (for limited charset).

### When to Use
- Count character frequencies
- Find most/least frequent element
- Compare if two strings are anagrams
- Group elements by some property

### Mental Model
```
Pass 1: Build frequency map
Pass 2: Use frequency map to answer query
```

### Generic Template
```java
public Map<Character, Integer> buildFrequencyMap(String s) {
    Map<Character, Integer> freq = new HashMap<>();

    for (char c : s.toCharArray()) {
        freq.put(c, freq.getOrDefault(c, 0) + 1);
    }

    return freq;
}

// For lowercase letters only (more efficient)
public int[] buildFrequencyArray(String s) {
    int[] freq = new int[26];

    for (char c : s.toCharArray()) {
        freq[c - 'a']++;
    }

    return freq;
}
```

### Example: Valid Anagram
```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;

    int[] freq = new int[26];

    for (int i = 0; i < s.length(); i++) {
        freq[s.charAt(i) - 'a']++;
        freq[t.charAt(i) - 'a']--;
    }

    for (int count : freq) {
        if (count != 0) return false;
    }

    return true;
}
```

### Example: Group Anagrams
```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();

    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);

        map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }

    return new ArrayList<>(map.values());
}
```

---

## Pattern 4: Index as Hash (Cycle Detection / Missing Numbers)

### Core Idea
When values are in range [1, n] or [0, n-1], use the value as an index to mark presence.

### When to Use
- Find missing/duplicate numbers in range [1, n]
- O(1) space constraint
- Values map naturally to indices

### Mental Model
```
for each value:
    use |value| as index
    mark nums[index] as negative (or modify somehow)

for each index:
    if nums[index] > 0 → index+1 is missing
```

### Generic Template
```java
public List<Integer> findMissing(int[] nums) {
    List<Integer> result = new ArrayList<>();

    // Mark presence by negating
    for (int i = 0; i < nums.length; i++) {
        int index = Math.abs(nums[i]) - 1;
        if (nums[index] > 0) {
            nums[index] = -nums[index];
        }
    }

    // Find unmarked indices
    for (int i = 0; i < nums.length; i++) {
        if (nums[i] > 0) {
            result.add(i + 1);
        }
    }

    return result;
}
```

### Example: Find All Duplicates in an Array
```java
public List<Integer> findDuplicates(int[] nums) {
    List<Integer> result = new ArrayList<>();

    for (int i = 0; i < nums.length; i++) {
        int index = Math.abs(nums[i]) - 1;

        if (nums[index] < 0) {
            result.add(index + 1);  // Already marked = duplicate
        } else {
            nums[index] = -nums[index];
        }
    }

    return result;
}
```

---

## Pattern 5: XOR for Single Element

### Core Idea
XOR cancels out pairs: a ^ a = 0, a ^ 0 = a. Use this to find unpaired elements.

### When to Use
- Find single number among pairs
- Find missing number
- Need O(1) space

### Mental Model
```
XOR all elements together
Pairs cancel out → only unpaired element remains
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
    int result = nums.length;  // Start with n

    for (int i = 0; i < nums.length; i++) {
        result ^= i ^ nums[i];  // XOR index and value
    }

    return result;
}
```

---

## Pattern 6: Boyer-Moore Voting Algorithm

### Core Idea
Find majority element (appears > n/2 times) in O(1) space by tracking a candidate and count.

### When to Use
- Find element appearing more than n/2 times
- Guaranteed majority exists
- O(1) space required

### Mental Model
```
maintain candidate and count
for each element:
    if count == 0 → new candidate
    if same as candidate → count++
    else → count--

majority always survives
```

### Example: Majority Element
```java
public int majorityElement(int[] nums) {
    int candidate = 0;
    int count = 0;

    for (int num : nums) {
        if (count == 0) {
            candidate = num;
        }

        count += (num == candidate) ? 1 : -1;
    }

    return candidate;
}
```

---

## Pattern 7: Kadane's Algorithm (Maximum Subarray)

### Core Idea
Track current subarray sum. If it goes negative, start fresh.

### When to Use
- Maximum/minimum subarray sum
- Contiguous subarray problems

### Mental Model
```
for each element:
    extend current subarray OR start new subarray
    current = max(num, current + num)
    update global max
```

### Example: Maximum Subarray
```java
public int maxSubArray(int[] nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];

    for (int i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }

    return maxSum;
}
```

---

## Pattern 8: Longest Consecutive Sequence (Set + Sequence Start)

### Core Idea
Only start counting from sequence beginnings (elements where num-1 doesn't exist).

### When to Use
- Find longest consecutive sequence
- O(n) time without sorting

### Mental Model
```
add all to set
for each element:
    if (num - 1) not in set → this is sequence start
        count consecutive elements
        update max length
```

### Example: Longest Consecutive Sequence
```java
public int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int num : nums) {
        set.add(num);
    }

    int maxLength = 0;

    for (int num : set) {
        // Only start if this is beginning of sequence
        if (!set.contains(num - 1)) {
            int currentNum = num;
            int length = 1;

            while (set.contains(currentNum + 1)) {
                currentNum++;
                length++;
            }

            maxLength = Math.max(maxLength, length);
        }
    }

    return maxLength;
}
```

---

## Quick Reference

| Pattern | Time | Space | Key Insight |
|---------|------|-------|-------------|
| HashMap Lookup | O(n) | O(n) | Complement exists in map? |
| HashSet Duplicate | O(n) | O(n) | add() returns false if exists |
| Frequency Count | O(n) | O(k) | int[26] for lowercase |
| Index as Hash | O(n) | O(1) | Negate to mark presence |
| XOR Single | O(n) | O(1) | Pairs cancel: a^a=0 |
| Boyer-Moore | O(n) | O(1) | Count survives for majority |
| Kadane's | O(n) | O(1) | max(num, current+num) |
| Consecutive Seq | O(n) | O(n) | Start only if num-1 absent |
