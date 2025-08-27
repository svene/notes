
# 🌿 Thymeleaf Cheat Sheet

A comprehensive reference for the most commonly used Thymeleaf expressions and utility objects. Designed for quick lookup while coding.

---

## 🔑 Expression Types

### 1. `${...}` – Variable Expressions
- Access values from the **Spring model** (controller `Model` attributes).
- Example:
```html
<span th:text="${user.name}">Name</span>
```
→ Displays the value of `user.getName()`.

---

### 2. `*{...}` – Selection Expressions
- Access properties **relative to a selection target** (`th:object`).  
- Works in forms but also in any element that defines a selection target.
- Examples:

**With a form:**
```html
<form th:object="${user}">
    <input type="text" th:field="*{email}" />
</form>
```

**Without a form:**
```html
<div th:object="${user}">
    <p th:text="*{name}"></p>
    <p th:text="*{email}"></p>
</div>
```
→ Equivalent to `${user.name}` and `${user.email}` but relative to the selection target.

---

### 3. `#{...}` – Message Expressions
- Lookup **localized messages** from `messages.properties`.
- Example:
```html
<label th:text="#{label.username}">Username</label>
```

---

### 4. `@{...}` – URL Expressions
- Build URLs that include the web application’s context path automatically.
- Example:
```html
<a th:href="@{/users/{id}(id=${user.id})}">View User</a>
```
→ Expands to `/users/42` if `user.id` is 42.

---

### 5. `~{...}` – Fragment Expressions
- Reuse template fragments (layouts, headers, footers, etc.).
- Example:
```html
<div th:replace="~{fragments/header :: title}"></div>
```
→ Inserts the `title` fragment from `fragments/header.html`.

---

## 🔧 Utility Objects (`#...`)

- **`#fields`** – Form binding and validation helpers
  ```html
  <div th:if="${#fields.hasErrors('email')}" th:errors="*{email}">Error</div>
  ```

- **`#dates`** – Date/time formatting helpers
  ```html
  <span th:text="${#dates.format(myDate, 'yyyy-MM-dd')}"></span>
  ```

- **`#numbers`** – Number formatting helpers
  ```html
  <span th:text="${#numbers.formatDecimal(price, 1, 'COMMA', 2, 'POINT')}"></span>
  ```

- **`#strings`** – String helpers
  ```html
  <span th:text="${#strings.isEmpty(name) ? 'N/A' : name}"></span>
  ```

- **`#lists`, `#sets`, `#maps`, `#arrays`** – Collection helpers
  ```html
  <li th:each="item : ${#lists.arrayList(items)}" th:text="${item}"></li>
  ```

- **`#temporals`** – Java 8+ time helpers (`LocalDate`, `Instant`, etc.)
  ```html
  <span th:text="${#temporals.format(myDateTime, 'yyyy-MM-dd HH:mm')}"></span>
  ```

---

## ✅ Key Takeaways

- **`${...}`** → Absolute expressions, from Spring model attributes.  
- **`*{...}`** → Relative expressions, evaluated against the current selection target (`th:object`). Not limited to forms.  
- **`#{...}`** → Localized message lookups.  
- **`@{...}`** → URL building with context path.  
- **`~{...}`** → Template fragments for reuse.  
- **`#...`** → Utility objects for validation, dates, numbers, strings, collections, and time.

Use `${...}` when accessing general model data, and `*{...}` when you want to work **relative to a selected object**, making templates cleaner and easier to maintain.
