# Linked List Patterns

Linked lists require careful pointer manipulation. Master these patterns for in-place operations.

---

## Pattern 1: Fast and Slow Pointers (Floyd's)

### Core Idea
Two pointers moving at different speeds. Used for cycle detection and finding middle.

### When to Use
- Detect cycle
- Find cycle start
- Find middle element
- Find kth element from end

### Mental Model
```
slow moves 1 step
fast moves 2 steps
if they meet → cycle exists
when fast reaches end → slow is at middle
```

### Example: Linked List Cycle
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow == fast) {
            return true;
        }
    }

    return false;
}
```

### Example: Middle of Linked List
```java
public ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }

    return slow;  // For even length, returns second middle
}
```

### Example: Find Cycle Start (Linked List Cycle II)
```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;

    // Find meeting point
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow == fast) {
            // Reset one pointer to head
            ListNode start = head;
            while (start != slow) {
                start = start.next;
                slow = slow.next;
            }
            return start;
        }
    }

    return null;
}
```

---

## Pattern 2: Reverse Linked List

### Core Idea
Change each node's next pointer to point backward.

### When to Use
- Reverse entire list
- Reverse portion of list
- Palindrome check

### Mental Model
```
prev = null
while curr != null:
    next = curr.next  (save next)
    curr.next = prev  (reverse)
    prev = curr       (move prev)
    curr = next       (move curr)
return prev
```

### Example: Reverse Linked List
```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode curr = head;

    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }

    return prev;
}

// Recursive version
public ListNode reverseListRecursive(ListNode head) {
    if (head == null || head.next == null) {
        return head;
    }

    ListNode newHead = reverseListRecursive(head.next);
    head.next.next = head;
    head.next = null;

    return newHead;
}
```

---

## Pattern 3: Two Pointers with Gap

### Core Idea
Maintain fixed gap between two pointers. When fast reaches end, slow is at target.

### When to Use
- Remove nth node from end
- Find kth element from end

### Mental Model
```
advance fast by n steps
move both until fast reaches end
slow is at target position
```

### Example: Remove Nth Node From End
```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0, head);
    ListNode slow = dummy, fast = dummy;

    // Advance fast by n + 1 steps
    for (int i = 0; i <= n; i++) {
        fast = fast.next;
    }

    // Move both until fast reaches end
    while (fast != null) {
        slow = slow.next;
        fast = fast.next;
    }

    // Skip the nth node
    slow.next = slow.next.next;

    return dummy.next;
}
```

---

## Pattern 4: Dummy Head Node

### Core Idea
Use a dummy node before head to simplify edge cases (head removal, empty list).

### When to Use
- Head might be modified
- Building new list
- Merging lists

### Mental Model
```
dummy = new ListNode(0)
dummy.next = head
// ... operations
return dummy.next
```

### Example: Merge Two Sorted Lists
```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;

    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            current.next = l1;
            l1 = l1.next;
        } else {
            current.next = l2;
            l2 = l2.next;
        }
        current = current.next;
    }

    current.next = (l1 != null) ? l1 : l2;

    return dummy.next;
}
```

---

## Pattern 5: In-Place Reversal of Sublist

### Core Idea
Reverse a portion of the list while maintaining connections.

### When to Use
- Reverse between positions
- Reverse in groups
- Reorder list

### Example: Reverse Nodes in k-Group
```java
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(0, head);
    ListNode prevGroupEnd = dummy;

    while (true) {
        // Check if k nodes exist
        ListNode kthNode = getKthNode(prevGroupEnd, k);
        if (kthNode == null) break;

        ListNode nextGroupStart = kthNode.next;
        ListNode groupStart = prevGroupEnd.next;

        // Reverse the group
        ListNode prev = nextGroupStart;
        ListNode curr = groupStart;
        while (curr != nextGroupStart) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }

        // Connect with previous part
        prevGroupEnd.next = kthNode;
        prevGroupEnd = groupStart;
    }

    return dummy.next;
}

private ListNode getKthNode(ListNode node, int k) {
    while (node != null && k > 0) {
        node = node.next;
        k--;
    }
    return node;
}
```

---

## Pattern 6: Find Middle + Reverse + Compare (Palindrome)

### Core Idea
Find middle, reverse second half, compare both halves.

### When to Use
- Check palindrome
- Compare two halves

### Example: Palindrome Linked List
```java
public boolean isPalindrome(ListNode head) {
    // Find middle
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }

    // Reverse second half
    ListNode secondHalf = reverse(slow);

    // Compare
    ListNode firstHalf = head;
    while (secondHalf != null) {
        if (firstHalf.val != secondHalf.val) {
            return false;
        }
        firstHalf = firstHalf.next;
        secondHalf = secondHalf.next;
    }

    return true;
}

private ListNode reverse(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;
        head.next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
```

---

## Pattern 7: Reorder List (Interleave)

### Core Idea
Find middle, reverse second half, interleave both halves.

### When to Use
- Reorder list alternating
- Interleave two lists

### Example: Reorder List
```java
public void reorderList(ListNode head) {
    if (head == null || head.next == null) return;

    // Find middle
    ListNode slow = head, fast = head;
    while (fast.next != null && fast.next.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }

    // Reverse second half
    ListNode second = reverse(slow.next);
    slow.next = null;  // Cut the list

    // Interleave
    ListNode first = head;
    while (second != null) {
        ListNode temp1 = first.next;
        ListNode temp2 = second.next;

        first.next = second;
        second.next = temp1;

        first = temp1;
        second = temp2;
    }
}
```

---

## Pattern 8: Merge K Sorted Lists (Heap)

### Core Idea
Use min-heap to always get the smallest node among k lists.

### When to Use
- Merge multiple sorted lists
- K-way merge

### Example: Merge K Sorted Lists
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(
        (a, b) -> a.val - b.val
    );

    // Add first node from each list
    for (ListNode node : lists) {
        if (node != null) {
            heap.offer(node);
        }
    }

    ListNode dummy = new ListNode(0);
    ListNode current = dummy;

    while (!heap.isEmpty()) {
        ListNode smallest = heap.poll();
        current.next = smallest;
        current = current.next;

        if (smallest.next != null) {
            heap.offer(smallest.next);
        }
    }

    return dummy.next;
}
```

---

## Pattern 9: Add Two Numbers

### Core Idea
Simulate digit-by-digit addition with carry.

### When to Use
- Add numbers represented as linked lists
- Handle carries

### Example: Add Two Numbers
```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    int carry = 0;

    while (l1 != null || l2 != null || carry > 0) {
        int sum = carry;

        if (l1 != null) {
            sum += l1.val;
            l1 = l1.next;
        }
        if (l2 != null) {
            sum += l2.val;
            l2 = l2.next;
        }

        carry = sum / 10;
        current.next = new ListNode(sum % 10);
        current = current.next;
    }

    return dummy.next;
}
```

---

## Quick Reference

| Pattern | Technique | Use Case |
|---------|-----------|----------|
| Fast/Slow | 2x speed difference | Cycle, middle, kth from end |
| Reverse | prev/curr/next dance | Reverse all or part |
| Gap Pointers | Fixed n-gap | Remove nth from end |
| Dummy Head | Placeholder node | Simplify edge cases |
| Sublist Reversal | Reverse portion | Reverse in groups |
| Middle + Reverse | Find mid, reverse half | Palindrome check |
| Interleave | Merge alternating | Reorder list |
| K-way Merge | Min-heap | Merge k sorted lists |

## Common Pitfalls

1. **Null pointer**: Always check `node != null` before accessing `node.next`
2. **Losing reference**: Save `next` before modifying pointers
3. **Cycle creation**: Ensure you're not accidentally creating cycles
4. **Off-by-one**: Be careful with middle element (odd vs even length)
