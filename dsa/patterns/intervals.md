# Intervals Patterns

Interval problems almost always start with sorting. Master these patterns for scheduling and range problems.

---

## Pattern 1: Merge Overlapping Intervals

### Core Idea
Sort by start time. Merge if current start <= previous end.

### When to Use
- Combine overlapping ranges
- Simplify interval list

### Mental Model
```
sort by start
for each interval:
    if overlaps with last merged → extend end
    else → add as new interval
```

### Example: Merge Intervals
```java
public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;

    // Sort by start time
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

    List<int[]> merged = new ArrayList<>();
    int[] current = intervals[0];
    merged.add(current);

    for (int[] interval : intervals) {
        if (interval[0] <= current[1]) {
            // Overlaps, extend end
            current[1] = Math.max(current[1], interval[1]);
        } else {
            // No overlap, add new interval
            current = interval;
            merged.add(current);
        }
    }

    return merged.toArray(new int[merged.size()][]);
}
```

---

## Pattern 2: Insert Interval

### Core Idea
Add intervals before new one, merge overlapping, add remaining.

### When to Use
- Insert into sorted interval list
- Maintain sorted and non-overlapping

### Mental Model
```
1. Add all intervals ending before new one starts
2. Merge all overlapping intervals with new one
3. Add remaining intervals
```

### Example: Insert Interval
```java
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0;
    int n = intervals.length;

    // Add all intervals ending before newInterval starts
    while (i < n && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i]);
        i++;
    }

    // Merge overlapping intervals
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);

    // Add remaining intervals
    while (i < n) {
        result.add(intervals[i]);
        i++;
    }

    return result.toArray(new int[result.size()][]);
}
```

---

## Pattern 3: Check for Overlap (Meeting Rooms)

### Core Idea
Sort by start time. Check if any meeting starts before previous ends.

### When to Use
- Detect conflicts
- Validate schedule

### Mental Model
```
sort by start
for each pair of consecutive intervals:
    if current.start < prev.end → conflict
```

### Example: Meeting Rooms (Can Attend All)
```java
public boolean canAttendMeetings(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] < intervals[i - 1][1]) {
            return false;  // Overlap found
        }
    }

    return true;
}
```

---

## Pattern 4: Minimum Meeting Rooms (Min Heap)

### Core Idea
Track end times with min-heap. Reuse room if it's free (end <= start).

### When to Use
- Count maximum concurrent events
- Minimum resources needed

### Mental Model
```
sort by start
for each meeting:
    if earliest ending room is free → reuse
    else → allocate new room
```

### Example: Meeting Rooms II
```java
public int minMeetingRooms(int[][] intervals) {
    if (intervals.length == 0) return 0;

    // Sort by start time
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

    // Min-heap of end times
    PriorityQueue<Integer> heap = new PriorityQueue<>();
    heap.offer(intervals[0][1]);

    for (int i = 1; i < intervals.length; i++) {
        // If earliest ending room is free
        if (intervals[i][0] >= heap.peek()) {
            heap.poll();  // Reuse this room
        }
        heap.offer(intervals[i][1]);  // Add current meeting's end time
    }

    return heap.size();
}
```

### Alternative: Line Sweep
```java
public int minMeetingRooms(int[][] intervals) {
    int[] starts = new int[intervals.length];
    int[] ends = new int[intervals.length];

    for (int i = 0; i < intervals.length; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }

    Arrays.sort(starts);
    Arrays.sort(ends);

    int rooms = 0, endPtr = 0;

    for (int i = 0; i < starts.length; i++) {
        if (starts[i] < ends[endPtr]) {
            rooms++;
        } else {
            endPtr++;
        }
    }

    return rooms;
}
```

---

## Pattern 5: Interval List Intersections

### Core Idea
Two pointers, one for each list. Find intersection, advance smaller end.

### When to Use
- Find common ranges between two interval lists
- Both lists are sorted

### Mental Model
```
intersection = [max(start1, start2), min(end1, end2)]
if valid (start <= end) → add to result
advance pointer with smaller end
```

### Example: Interval List Intersections
```java
public int[][] intervalIntersection(int[][] A, int[][] B) {
    List<int[]> result = new ArrayList<>();
    int i = 0, j = 0;

    while (i < A.length && j < B.length) {
        int start = Math.max(A[i][0], B[j][0]);
        int end = Math.min(A[i][1], B[j][1]);

        if (start <= end) {
            result.add(new int[] { start, end });
        }

        // Advance pointer with smaller end
        if (A[i][1] < B[j][1]) {
            i++;
        } else {
            j++;
        }
    }

    return result.toArray(new int[result.size()][]);
}
```

---

## Pattern 6: Remove Covered Intervals

### Core Idea
Sort by start (ascending), then by end (descending). Count intervals not covered.

### When to Use
- Remove redundant intervals
- Find non-covered intervals

### Mental Model
```
sort: by start ASC, then by end DESC
if current.end > maxEnd → not covered, count it
update maxEnd
```

### Example: Remove Covered Intervals
```java
public int removeCoveredIntervals(int[][] intervals) {
    // Sort by start ASC, then by end DESC
    Arrays.sort(intervals, (a, b) ->
        a[0] != b[0] ? a[0] - b[0] : b[1] - a[1]
    );

    int count = 0;
    int maxEnd = 0;

    for (int[] interval : intervals) {
        if (interval[1] > maxEnd) {
            count++;
            maxEnd = interval[1];
        }
    }

    return count;
}
```

---

## Pattern 7: Employee Free Time

### Core Idea
Merge all busy intervals, gaps are free time.

### When to Use
- Find gaps in schedule
- Common free time across multiple schedules

### Example: Employee Free Time
```java
public List<Interval> employeeFreeTime(List<List<Interval>> schedule) {
    // Collect all intervals
    List<Interval> all = new ArrayList<>();
    for (List<Interval> emp : schedule) {
        all.addAll(emp);
    }

    // Sort by start
    Collections.sort(all, (a, b) -> a.start - b.start);

    List<Interval> result = new ArrayList<>();
    int prevEnd = all.get(0).end;

    for (Interval interval : all) {
        if (interval.start > prevEnd) {
            // Gap found
            result.add(new Interval(prevEnd, interval.start));
        }
        prevEnd = Math.max(prevEnd, interval.end);
    }

    return result;
}
```

---

## Pattern 8: My Calendar (Interval Conflict Check)

### Core Idea
Check if new interval overlaps with any existing interval.

### When to Use
- Booking system
- Detect double booking

### Mental Model
```
two intervals [s1, e1] and [s2, e2] overlap if:
    s1 < e2 AND s2 < e1
```

### Example: My Calendar
```java
class MyCalendar {
    private List<int[]> bookings;

    public MyCalendar() {
        bookings = new ArrayList<>();
    }

    public boolean book(int start, int end) {
        for (int[] booking : bookings) {
            // Check overlap: s1 < e2 && s2 < e1
            if (start < booking[1] && booking[0] < end) {
                return false;
            }
        }
        bookings.add(new int[] { start, end });
        return true;
    }
}
```

---

## Quick Reference

| Pattern | Sort By | Key Operation |
|---------|---------|---------------|
| Merge | Start | Extend end if overlap |
| Insert | Already sorted | Add before, merge, add after |
| Check Overlap | Start | curr.start < prev.end |
| Min Rooms | Start | Min-heap of end times |
| Intersections | Two pointers | max(starts), min(ends) |
| Remove Covered | Start ASC, End DESC | Track maxEnd |
| Free Time | Start | Find gaps |
| Calendar | - | s1 < e2 && s2 < e1 |

## Overlap Conditions

```
Two intervals [s1, e1] and [s2, e2]:

Overlap:     s1 < e2 && s2 < e1
             (neither ends before other starts)

No overlap:  e1 <= s2 || e2 <= s1
             (one ends before other starts)
```

## When to Use Each Sort

- **Sort by start**: Merge, overlap detection, most problems
- **Sort by end**: Greedy interval scheduling (max non-overlapping)
- **Sort by both**: Remove covered intervals
