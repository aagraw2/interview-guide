# Interview Guide

A collection of resources I put together for software engineering interview prep. Covers DSA, System Design (HLD), and Low Level Design (LLD).

## Table of Contents

- [What is this?](#what-is-this)
- [Repository Structure](#repository-structure)
- [DSA](#dsa)
- [High Level Design](#high-level-design)
- [Low Level Design](#low-level-design)
- [How to Use](#how-to-use)
- [Prerequisites](#prerequisites)

## What is this?

This repo is my attempt to organize everything needed to prepare for technical interviews. Instead of jumping around between random problems, I grouped questions by patterns so you can focus on mastering one concept at a time.

The DSA section has a practice sheet with 150+ problems organized by topic and priority (P1 = must know, P2 = good to know, P3 = nice to have). Each topic also has a detailed patterns guide that explains the mental models, templates, and when to apply each technique.

The HLD and LLD sections contain concept explanations and example system designs that you can study and reference.

## Repository Structure

```
interview-guide/
├── dsa/
│   ├── DSA Practice Sheet.md      # 150+ problems by topic with priority levels
│   └── patterns/                   # Pattern guides for each topic
│       ├── arrays-hashing.md
│       ├── sliding-window.md
│       ├── two-pointers.md
│       ├── binary-search.md
│       ├── stack.md
│       ├── linked-list.md
│       ├── trees.md
│       ├── graphs.md
│       ├── dynamic-programming.md
│       ├── backtracking.md
│       ├── greedy.md
│       ├── intervals.md
│       ├── matrix.md
│       ├── heap-trie.md
│       ├── prefix-sums.md
│       └── bit-manipulation.md
├── hld/
│   ├── concepts/                   # HLD concepts (caching, load balancing, etc.)
│   └── example-systems/            # Example system designs (URL shortener, etc.)
├── lld/
│   ├── design-patterns/            # SOLID, design patterns
│   └── example-systems/            # LLD examples (parking lot, elevator, etc.)
├── DSA Interviews.md
├── High Level Design Interviews.md
└── Low Level Design Interviews.md
```

## DSA

### Practice Sheet

The [DSA Practice Sheet](dsa/DSA%20Practice%20Sheet.md) contains problems organized by topic:

- Arrays & Hashing
- Prefix Sums
- Sliding Window
- Two Pointers
- Binary Search
- Stack
- Linked List
- Greedy
- Intervals
- Backtracking
- Dynamic Programming
- Trees
- Graphs
- Matrix
- Heap / Trie
- Bit Manipulation

Each problem has:
- Link to LeetCode/GFG
- Core concept being tested
- Priority level (P1/P2/P3)
- Difficulty

### Pattern Guides

The `dsa/patterns/` folder contains detailed guides for each topic. Each guide covers:

- Different patterns within that topic
- How to identify when to use each pattern
- Mental model and template code
- Solved examples in Java
- Quick reference tables

For example, the sliding window guide covers fixed size windows, variable size (expand/shrink), at most K distinct, minimum window substring, and more.

## High Level Design

See [High Level Design Interviews.md](High%20Level%20Design%20Interviews.md) for the overview.

The `hld/` folder contains:
- `concepts/` - Load balancing, caching, databases, message queues, etc.
- `example-systems/` - Full system design examples

## Low Level Design

See [Low Level Design Interviews.md](Low%20Level%20Design%20Interviews.md) for the overview.

The `lld/` folder contains:
- `design-patterns/` - SOLID principles, common design patterns
- `example-systems/` - Parking lot, elevator system, and other classic LLD problems

## How to Use

### Option 1: Obsidian (Recommended)

This repo works best with [Obsidian](https://obsidian.md/). Just open the folder as a vault and you get:
- Linked notes and easy navigation
- Graph view to see connections
- Better markdown rendering

### Option 2: Any Markdown Viewer

You can also just browse the markdown files directly on GitHub or in VS Code.

### Study Approach

1. **Start with patterns** - Before grinding problems, read through the pattern guide for a topic. Understand the templates and when to use them.

2. **Do P1 problems first** - These are the most commonly asked. Master these before moving to P2/P3.

3. **Time yourself** - For easy problems aim for 15 min, medium 25 min, hard 40 min. If stuck, study the solution and come back to it later.

4. **Review weak areas** - Track which patterns trip you up and revisit those guides.

## Prerequisites

Before diving into problems, make sure you're comfortable with:

**Data Structures**
- Arrays and Strings
- HashMaps and HashSets
- Linked Lists
- Stacks and Queues
- Trees (Binary Tree, BST)
- Graphs (adjacency list, matrix)
- Heaps / Priority Queues

**Algorithms**
- Recursion
- BFS and DFS
- Binary Search
- Sorting (know at least one O(n log n) sort)

**Language Basics**

If using Java, know these well:
- `Arrays.sort()`, `Arrays.fill()`, `Arrays.copyOf()`
- `String` methods: `substring()`, `toCharArray()`, `charAt()`
- `Collections`: `ArrayList`, `HashMap`, `HashSet`, `PriorityQueue`
- `Math.max()`, `Math.min()`, `Math.abs()`

---

Good luck with your prep.
