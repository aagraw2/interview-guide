# Stack Patterns

Stacks provide LIFO (Last In, First Out) processing. Essential for matching, parsing, and monotonic problems.

---

## Pattern 1: Matching Brackets / Parentheses

### Core Idea
Push opening brackets, pop on closing brackets and check for match.

### When to Use
- Validate parentheses/brackets
- Match opening and closing pairs
- Nested structures

### Mental Model
```
for each char:
    if opening → push
    if closing → pop and check match
stack should be empty at end
```

### Example: Valid Parentheses
```java
public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    Map<Character, Character> pairs = Map.of(
        ')', '(',
        ']', '[',
        '}', '{'
    );

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) {
            stack.push(c);
        } else {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false;
            }
        }
    }

    return stack.isEmpty();
}
```

---

## Pattern 2: Monotonic Stack (Next Greater Element)

### Core Idea
Maintain stack in monotonic order (increasing/decreasing). When violated, pop and process.

### When to Use
- Next greater/smaller element
- Previous greater/smaller element
- Stock span, daily temperatures

### Mental Model
```
Monotonic Decreasing Stack (for next greater):
for each element:
    while stack not empty AND stack.top < current:
        pop → current is next greater for popped
    push current
```

### Generic Template
```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Stack<Integer> stack = new Stack<>();  // Store indices

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }

    return result;
}
```

### Example: Daily Temperatures
```java
public int[] dailyTemperatures(int[] temperatures) {
    int n = temperatures.length;
    int[] result = new int[n];
    Stack<Integer> stack = new Stack<>();  // Store indices

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temperatures[stack.peek()] < temperatures[i]) {
            int prevIndex = stack.pop();
            result[prevIndex] = i - prevIndex;
        }
        stack.push(i);
    }

    return result;
}
```

---

## Pattern 3: Monotonic Stack (Previous Smaller Element)

### Core Idea
Find previous smaller element by maintaining increasing stack.

### Mental Model
```
Monotonic Increasing Stack (for previous smaller):
for each element:
    while stack not empty AND stack.top >= current:
        pop
    stack.top is previous smaller (or -1 if empty)
    push current
```

### Example: Stock Span Problem
```java
class StockSpanner {
    Stack<int[]> stack;  // [price, span]

    public StockSpanner() {
        stack = new Stack<>();
    }

    public int next(int price) {
        int span = 1;

        while (!stack.isEmpty() && stack.peek()[0] <= price) {
            span += stack.pop()[1];
        }

        stack.push(new int[] { price, span });
        return span;
    }
}
```

---

## Pattern 4: Evaluate Expression (Postfix / Calculator)

### Core Idea
Push operands to stack. On operator, pop operands, compute, push result.

### When to Use
- Evaluate reverse polish notation
- Basic calculator
- Expression parsing

### Example: Evaluate Reverse Polish Notation
```java
public int evalRPN(String[] tokens) {
    Stack<Integer> stack = new Stack<>();
    Set<String> operators = Set.of("+", "-", "*", "/");

    for (String token : tokens) {
        if (operators.contains(token)) {
            int b = stack.pop();
            int a = stack.pop();

            switch (token) {
                case "+": stack.push(a + b); break;
                case "-": stack.push(a - b); break;
                case "*": stack.push(a * b); break;
                case "/": stack.push(a / b); break;
            }
        } else {
            stack.push(Integer.parseInt(token));
        }
    }

    return stack.pop();
}
```

---

## Pattern 5: Decode Nested Patterns

### Core Idea
Use stack to handle nested structures. Push state when entering nested level, pop when exiting.

### When to Use
- Decode strings with nested brackets
- Nested structures with multipliers

### Mental Model
```
on '[': save current state to stack, reset
on ']': pop state, combine with current
on digit: update multiplier
on letter: add to current string
```

### Example: Decode String
```java
public String decodeString(String s) {
    Stack<StringBuilder> strStack = new Stack<>();
    Stack<Integer> numStack = new Stack<>();
    StringBuilder current = new StringBuilder();
    int num = 0;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            num = num * 10 + (c - '0');
        } else if (c == '[') {
            strStack.push(current);
            numStack.push(num);
            current = new StringBuilder();
            num = 0;
        } else if (c == ']') {
            StringBuilder prev = strStack.pop();
            int repeat = numStack.pop();
            for (int i = 0; i < repeat; i++) {
                prev.append(current);
            }
            current = prev;
        } else {
            current.append(c);
        }
    }

    return current.toString();
}
```

---

## Pattern 6: Min Stack (Track Minimum)

### Core Idea
Store minimum along with each element, or use auxiliary stack.

### When to Use
- Need O(1) getMin operation
- Stack with minimum tracking

### Example: Min Stack
```java
class MinStack {
    private Stack<int[]> stack;  // [value, minSoFar]

    public MinStack() {
        stack = new Stack<>();
    }

    public void push(int val) {
        int min = stack.isEmpty() ? val : Math.min(val, stack.peek()[1]);
        stack.push(new int[] { val, min });
    }

    public void pop() {
        stack.pop();
    }

    public int top() {
        return stack.peek()[0];
    }

    public int getMin() {
        return stack.peek()[1];
    }
}
```

---

## Pattern 7: Largest Rectangle in Histogram

### Core Idea
Use monotonic increasing stack. When smaller bar found, calculate areas for popped bars.

### When to Use
- Largest rectangle problems
- Area calculations with heights

### Mental Model
```
maintain increasing stack of indices
when smaller bar found:
    pop and calculate area
    width = current_index - stack.top - 1
    area = height[popped] * width
```

### Example: Largest Rectangle in Histogram
```java
public int largestRectangleArea(int[] heights) {
    Stack<Integer> stack = new Stack<>();
    int maxArea = 0;
    int n = heights.length;

    for (int i = 0; i <= n; i++) {
        int currentHeight = (i == n) ? 0 : heights[i];

        while (!stack.isEmpty() && heights[stack.peek()] > currentHeight) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }

        stack.push(i);
    }

    return maxArea;
}
```

---

## Pattern 8: Asteroid Collision

### Core Idea
Simulate collisions using stack. Push when no collision, resolve when opposite directions meet.

### When to Use
- Simulation with cancellation
- Objects moving in opposite directions

### Example: Asteroid Collision
```java
public int[] asteroidCollision(int[] asteroids) {
    Stack<Integer> stack = new Stack<>();

    for (int asteroid : asteroids) {
        boolean exploded = false;

        while (!stack.isEmpty() && asteroid < 0 && stack.peek() > 0) {
            if (stack.peek() < -asteroid) {
                stack.pop();  // Stack asteroid explodes
                continue;
            } else if (stack.peek() == -asteroid) {
                stack.pop();  // Both explode
            }
            exploded = true;
            break;
        }

        if (!exploded) {
            stack.push(asteroid);
        }
    }

    int[] result = new int[stack.size()];
    for (int i = result.length - 1; i >= 0; i--) {
        result[i] = stack.pop();
    }
    return result;
}
```

---

## Quick Reference

| Pattern | Stack Type | When to Pop | Use Case |
|---------|------------|-------------|----------|
| Matching | Any | Closing bracket | Validate parentheses |
| Next Greater | Mono Decreasing | Current > top | Daily temperatures |
| Previous Smaller | Mono Increasing | Current <= top | Stock span |
| Evaluate | Operand stack | On operator | Calculator |
| Decode Nested | Two stacks | On ']' | Decode string |
| Min Stack | Store min with element | Normal pop | O(1) getMin |
| Histogram | Mono Increasing | Smaller height | Largest rectangle |
| Collision | Simulation | Opposite directions | Asteroid collision |

## When to Use Stack

- Matching pairs (brackets, tags)
- Nested structures
- Previous/next greater/smaller element
- Expression evaluation
- Undo operations
- DFS (iterative)
