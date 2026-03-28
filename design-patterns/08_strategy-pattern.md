# Strategy Pattern

## Intent
The Strategy pattern allows selecting an algorithm at runtime by encapsulating them in separate classes.

## When to Use
- Multiple algorithms
- Switch behavior dynamically
- Avoid conditionals

## Example
```java
public interface Sortable implements Sortable {
    int[] sort(int[] arr);
}

public class MergeSort implements Sortable {
    @Override
    public int[] sort(int[] arr){
        System.out.println("Sorting using Merge Sort");
        return arr;
    }
}

public class QuickSort implements Sortable {
    @Override
    public int[] sort(int[] arr){
        System.out.println("Sorting using Quick Sort");
        return arr;
    }    
}

public class SortContext {

    private Sortable strategy;

    public SortContext(Sortable strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(Sortable strategy) {
        this.strategy = strategy;
    }

    public int[] sort(int[] arr) {
        return strategy.sort(arr);
    }
}

public class Main {

    public static void main(String[] args) {

        int[] arr = {5,2,9,1};

        SortContext context = new SortContext(new MergeSort());
        context.sort(arr);

        context.setStrategy(new QuickSort());
        context.sort(arr);
    }
}
```
