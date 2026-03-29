
# 1. Functional Requirements

- Enable/disable features dynamically
    
- Support user-based feature rollout
    
- Support percentage rollout (e.g., 10% users)
    
- Support environment-based toggles (dev, prod)
    
- Support feature dependencies
    
- Fetch feature status at runtime
    
- Update toggles without redeploy
    

---

# 2. Non-Functional Requirements

- Low latency evaluation
    
- High availability
    
- Scalable
    
- Thread-safe
    
- Consistent evaluation logic
    
- Extensible for new rollout strategies
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|Feature|Represents a toggleable feature|
|FeatureStatus|ON / OFF|
|UserContext|User-specific data|
|Rule|Condition to evaluate feature|
|Strategy|Rollout logic|
|FeatureRepository|Stores feature configs|
|FeatureService|Evaluates feature state|

---

# 4. Flows

---

## Evaluate Feature

1. Client requests feature status
    
2. Fetch feature config
    
3. Evaluate rules
    
4. Apply strategy
    
5. Return ON/OFF
    

---

## Update Feature

1. Admin updates feature config
    
2. Store in repository
    
3. New config used immediately
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUM
enum FeatureStatus {
    ON, OFF
}

// MODELS
class Feature {
    String name;
    FeatureStatus defaultStatus;
    List<Rule> rules;

    public Feature(String name, FeatureStatus status, List<Rule> rules) {
        this.name = name;
        this.defaultStatus = status;
        this.rules = rules;
    }
}

class UserContext {
    String userId;
    String country;

    public UserContext(String userId, String country) {
        this.userId = userId;
        this.country = country;
    }
}

// RULE
interface Rule {
    boolean evaluate(UserContext ctx);
}

// RULE: Country based
class CountryRule implements Rule {
    private String allowedCountry;

    public CountryRule(String country) {
        this.allowedCountry = country;
    }

    public boolean evaluate(UserContext ctx) {
        return allowedCountry.equals(ctx.country);
    }
}

// STRATEGY: Rollout
interface RolloutStrategy {
    boolean isEnabled(UserContext ctx);
}

// Percentage rollout
class PercentageStrategy implements RolloutStrategy {
    private int percentage;

    public PercentageStrategy(int percentage) {
        this.percentage = percentage;
    }

    public boolean isEnabled(UserContext ctx) {
        int hash = Math.abs(ctx.userId.hashCode() % 100);
        return hash < percentage;
    }
}

// User whitelist rollout
class UserWhitelistStrategy implements RolloutStrategy {
    private Set<String> users;

    public UserWhitelistStrategy(Set<String> users) {
        this.users = users;
    }

    public boolean isEnabled(UserContext ctx) {
        return users.contains(ctx.userId);
    }
}

// REPOSITORY
class FeatureRepository {
    private Map<String, FeatureConfig> store = new ConcurrentHashMap<>();

    public void save(String name, FeatureConfig config) {
        store.put(name, config);
    }

    public FeatureConfig get(String name) {
        return store.get(name);
    }
}

// CONFIG (combines rules + strategy)
class FeatureConfig {
    FeatureStatus defaultStatus;
    List<Rule> rules;
    RolloutStrategy strategy;

    public FeatureConfig(FeatureStatus status,
                         List<Rule> rules,
                         RolloutStrategy strategy) {
        this.defaultStatus = status;
        this.rules = rules;
        this.strategy = strategy;
    }
}

// SERVICE
class FeatureService {
    private FeatureRepository repo;

    public FeatureService(FeatureRepository repo) {
        this.repo = repo;
    }

    public boolean isEnabled(String featureName, UserContext ctx) {
        FeatureConfig config = repo.get(featureName);

        if (config == null) return false;

        // Evaluate rules
        for (Rule rule : config.rules) {
            if (!rule.evaluate(ctx)) {
                return false;
            }
        }

        // Apply rollout strategy
        if (config.strategy != null) {
            return config.strategy.isEnabled(ctx);
        }

        return config.defaultStatus == FeatureStatus.ON;
    }
}
```

---

# 6. Complexity Analysis

- Fetch config → O(1)
    
- Rule evaluation → O(R)
    
- Strategy evaluation → O(1)
    

**Overall: O(R)**

---

# 7. Design Patterns Used

- Strategy Pattern → Rollout strategies
    
- Strategy Pattern → Rule evaluation
    
- Repository Pattern → Feature storage
    
- Dependency Injection → Loose coupling
    

---

# 8. Key Design Decisions

- Rules and rollout strategies separated for flexibility
    
- Deterministic hashing for percentage rollout
    
- Default fallback when no config present
    
- Thread-safe storage using ConcurrentHashMap
    
- Feature evaluation kept lightweight for low latency
    
- Config-driven design enables dynamic updates
    

---

# 9. Edge Cases Handled

- Feature not found
    
- Empty rules list
    
- Conflicting rule evaluations
    
- Consistent user bucketing for rollout
    

---

# 10. Possible Extensions

- Add dependency between features
    
- Add time-based rollout (start/end time)
    
- Use distributed cache (Redis) for configs
    
- Add admin dashboard
    
- Add audit logs for changes
    
- Support A/B testing
    
- Add hierarchical rules (AND/OR logic)