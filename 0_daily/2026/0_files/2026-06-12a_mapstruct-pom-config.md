# Maven & MapStruct POM Configuration — Summary

## 1. `annotationProcessorPaths` — MapStruct Processor

```xml
<annotationProcessorPaths>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${org.mapstruct.version}</version>
    </path>
</annotationProcessorPaths>
```

Tells the Maven compiler plugin to use MapStruct's annotation processor at compile time.
It scans for `@Mapper` and `@Mapping` annotations and **auto-generates mapper implementation classes**
(e.g. `CarMapperImpl.java`) so you don't have to write boilerplate mapping code.

Using `annotationProcessorPaths` puts the processor on a **dedicated classpath**, separate from
your regular dependencies — avoiding version conflicts and keeping it out of your final artifact.

Without it, no implementation is generated and you'll get a `NullPointerException` or
`NoSuchBeanDefinitionException` at runtime.

---

## 2. `mapstruct-processor` as a `<dependency>` — Not Needed

```xml
<!-- NOT needed if annotationProcessorPaths is configured -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>${org.mapstruct.version}</version>
    <scope>provided</scope>
</dependency>
```

This is the **old approach** — it puts the processor on the main classpath (`provided` just prevents
it from being bundled in the jar). Having both is redundant and can cause the processor to run twice.

**Keep only the `annotationProcessorPaths` entry.** You still need the core MapStruct API dependency:

```xml
<!-- Keep this — annotations and interfaces your code references -->
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
</dependency>
```

---

## 3. `<showWarnings>true</showWarnings>`

```xml
<showWarnings>true</showWarnings>
```

Tells the Maven compiler plugin to **print `javac` warnings to the console** during the build.
By default Maven suppresses them. Enabling this surfaces:

- **Unchecked/unsafe operations** — raw types, unchecked casts
- **Deprecation warnings** — use of deprecated APIs
- **MapStruct warnings** — unmapped properties, ambiguous mappings, e.g.:

```
warning: Unmapped target property: "firstName".
warning: Unmapped source property: "lastName".
```

Especially useful in a MapStruct project where silent unmapped fields can cause subtle runtime bugs.
