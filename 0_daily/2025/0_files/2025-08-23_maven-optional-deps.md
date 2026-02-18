# Optional Dependencies Pattern: `mylib` / `myapp` Example

## `mylib`

**POM configuration (optional dependencies for Jackson and Gson)**

```xml
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.2</version>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.10.1</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

** Runtime detection**

```java
public class JsonProvider {
private static boolean jacksonAvailable;
private static boolean gsonAvailable;

    static {
        try { Class.forName("com.fasterxml.jackson.databind.ObjectMapper"); jacksonAvailable = true; } 
        catch (ClassNotFoundException e) { jacksonAvailable = false; }
        try { Class.forName("com.google.gson.Gson"); gsonAvailable = true; } 
        catch (ClassNotFoundException e) { gsonAvailable = false; }
    }

    public static void serialize(Object obj) {
        if (jacksonAvailable) {
            // use Jackson
        } else if (gsonAvailable) {
            // use Gson
        } else {
            throw new IllegalStateException("No JSON library found");
        }
    }
}
```

Key Points:
- mylib compiles without forcing either Jackson or Gson onto consumers.
- Consumers can choose which library to include at runtime.
- Optional dependencies are not transitive by default.


## `myapp`

If myapp wants to use mylib with Jackson, it must explicitly declare Jackson as a runtime dependency:

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>mylib</artifactId>
        <version>1.0.0</version>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.2</version>
        <scope>runtime</scope> <!-- optional: compile scope if myapp uses Jackson directly -->
    </dependency>
</dependencies>
```



## Summary of the Pattern
- mylib declares optional dependencies: “I can use these, but consumers don’t have to pull them in.”
- At runtime, mylib detects which library is available (reflection / Class.forName).
- myapp decides which optional dependency to provide at runtime.
- This keeps the library flexible, modular, and avoids forcing unnecessary dependencies downstream.
