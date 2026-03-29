
## Intent
The Decorator pattern adds behavior to objects dynamically without modifying the original class.

## When to Use
- Add features at runtime
- Avoid subclass explosion (too many subclasses)
- Flexible combinations

## Example
```java
public interface UIComponent {
    void render();
}

public class TextBox implements UIComponent {
    @Override
    public void render() {
        System.out.println("Rendering TextBox");
    }
}

public abstract class ComponentDecorator implements UIComponent {

    protected UIComponent component;

    public ComponentDecorator(UIComponent component) {
        this.component = component;
    }

    @Override
    public void render() {
        component.render();
    }
}

public class BorderDecorator extends ComponentDecorator {

    public BorderDecorator(UIComponent component) {
        super(component);
    }

    @Override
    public void render() {
        super.render();
        addBorder();
    }

    private void addBorder() {
        System.out.println("Adding Border");
    }
}

public class Main {

    public static void main(String[] args) {

        UIComponent textBox =
                new ShadowDecorator(
                    new BorderDecorator(
                        new ScrollDecorator(
                            new TextBox()
                        )
                    )
                );

        textBox.render();
    }
}
```
