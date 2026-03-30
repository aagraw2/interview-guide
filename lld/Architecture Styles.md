
## Overview
Quick interview-friendly comparison of common architecture styles: Layered, Clean, and Hexagonal (Ports & Adapters).

## 1. Layered Architecture (Traditional)

### Structure

```
Controller → Service → Repository → Database
```

### Key Idea

- Organize code by **technical layers**
    

### Dependency Direction

- Top → Down (Controller depends on Service, Service on Repository)
    

### Characteristics

- Simple and widely used
    
- Easy to understand
    
- Tight coupling between layers
    

### Problem

- Business logic depends on infrastructure (DB, frameworks)
    
- Hard to test core logic independently
    

---

## 2. Clean Architecture

### Structure

```
Entities (Core)
↓
Use Cases (Business Logic)
↓
Interface Adapters
↓
Frameworks & Drivers (DB, UI)
```

### Key Idea

- **Dependency Inversion** → Inner layers should NOT depend on outer layers
    

### Dependency Direction

- Always **inward**
    

### Characteristics

- Core business logic is independent
    
- Highly testable
    
- Frameworks are replaceable
    

### Focus

- Separation of **business rules from infrastructure**
    

---

## 3. Hexagonal Architecture (Ports & Adapters)

### Structure

```
        External Systems
             ↓
   Adapter → Port → Core Domain ← Port ← Adapter
             ↑
        External Systems
```

### Key Idea

- Core business logic interacts via **Ports (interfaces)**
    
- External systems connect via **Adapters**
    

### Dependency Direction

- External → Core via interfaces
    
- Core does NOT depend on external systems
    

### Characteristics

- Highly decoupled
    
- Easy to plug/unplug external systems (DB, APIs)
    
- Great for testing and extensibility
    

---

## 4. Key Differences

|Aspect|Layered|Clean|Hexagonal|
|---|---|---|---|
|Organization|Technical layers|Business-centric layers|Ports & adapters|
|Dependency Direction|Top → Down|Inward|Inward|
|Business Logic|Mixed with layers|Centralized|Centralized|
|Coupling|High|Low|Very Low|
|Testability|Moderate|High|Very High|
|Flexibility|Low|High|Very High|

---

## 5. Simple Intuition

- **Layered** → “Stack of layers”
    
- **Clean** → “Onion (core inside, dependencies inward)”
    
- **Hexagonal** → “Core with plug-in adapters”
    

---

## 6. When to Use

- **Layered**
    
    - Small to medium systems
        
    - Quick development
        
    - Low complexity
        
- **Clean**
    
    - Complex business logic
        
    - Long-term maintainability
        
    - Strong separation needed
        
- **Hexagonal**
    
    - Systems with many integrations (DB, APIs, queues)
        
    - Need for high testability
        
    - Frequent change in external systems
        

---

## 7. Important Insight (Interview Gold)

- **Clean and Hexagonal are conceptually similar**
    
- Both:
    
    - Protect the **core business logic**
        
    - Use **interfaces to invert dependencies**
        

👉 Difference:

- **Clean** → focuses on _layered separation of concerns_
    
- **Hexagonal** → focuses on _interaction via ports & adapters_

---

## Summary
- **Layered**: simplest, but tighter coupling across layers.
- **Clean**: protects business core via dependency inversion (more testable).
- **Hexagonal**: business core talks through **ports**, with adapters for external systems.