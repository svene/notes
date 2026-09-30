# Java Records: Custom Constructors vs. Canonical Constructors

## User question

> for java records: difference between custom constructor and canonical constructors

## Answer

In Java records, there are three related concepts:

1. **Implicit canonical constructor**
2. **Explicit canonical constructor**
3. **Custom (non-canonical) constructors**

Example record:

```java
public record Person(String name, int age) {
}
```

The record components are:

```java
String name
int age
```

### 1. Implicit canonical constructor

If you do not write a constructor, Java generates this automatically:

```java
public Person(String name, int age) {
    this.name = name;
    this.age = age;
}
```

This is called the **canonical constructor** because its parameters exactly match the record components.

### 2. Explicit canonical constructor

You can define the canonical constructor yourself.

There are two forms.

#### Full canonical constructor

```java
public record Person(String name, int age) {

    public Person(String name, int age) {
        if (age < 0) {
            throw new IllegalArgumentException();
        }

        this.name = name;
        this.age = age;
    }
}
```

Characteristics:

- parameter list must exactly match the record components
- you must assign all fields yourself
- useful for validation/transformation

#### Compact canonical constructor

More idiomatic for records:

```java
public record Person(String name, int age) {

    public Person {
        if (age < 0) {
            throw new IllegalArgumentException();
        }

        name = name.trim();
    }
}
```

Java automatically inserts:

```java
this.name = name;
this.age = age;
```

at the end.

Characteristics:

- no parameter list
- parameters are implicitly available
- you cannot write field assignments manually
- best choice for validation/normalization

### 3. Custom (non-canonical) constructors

These are additional constructors with different parameter lists.

Example:

```java
public record Person(String name, int age) {

    public Person(String name) {
        this(name, 0);
    }
}
```

Characteristics:

- parameter list differs from record components
- must delegate to another constructor using `this(...)`
- eventually must reach the canonical constructor

This is similar to telescoping constructors in normal classes.

### Important rule

A non-canonical constructor may **not** assign fields directly.

This is illegal:

```java
public Person(String name) {
    this.name = name; // ERROR
}
```

because record fields are final and controlled by the canonical constructor.

You must do:

```java
public Person(String name) {
    this(name, 0);
}
```

### Practical guideline

#### Use compact canonical constructor for:

- validation
- normalization
- invariants

Example:

```java
public record Email(String value) {

    public Email {
        value = value.toLowerCase().trim();

        if (!value.contains("@")) {
            throw new IllegalArgumentException();
        }
    }
}
```

#### Use additional custom constructors for convenience APIs

Example:

```java
public record HostPort(String host, int port) {

    public HostPort(String host) {
        this(host, 8080);
    }
}
```

### Summary

| Type | Parameter list | Assign fields manually | Typical usage |
|---|---|---|---|
| Implicit canonical | same as components | generated automatically | default |
| Explicit canonical | same as components | yes (full form) | validation/transformation |
| Compact canonical | same as components (implicit) | no | preferred validation style |
| Custom constructor | different | no, must delegate | convenience overloads |
