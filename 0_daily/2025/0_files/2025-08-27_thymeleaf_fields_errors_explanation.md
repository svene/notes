<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Thymeleaf #fields.hasErrors Explanation</title>
</head>
<body>
<h1>Thymeleaf <code>#fields.hasErrors</code> Explanation</h1>

<p>In Thymeleaf, when you write:</p>
<pre><code>th:if="${#fields.hasErrors('email')}"
</code></pre>

<p>the <code>#fields</code> utility refers <strong>implicitly to the current BindingResult object</strong> associated with the form’s backing object.</p>

<h2>Example in Spring MVC:</h2>
<pre><code>@PostMapping("/register")
public String register(@Valid @ModelAttribute("user") UserDto userDto,
                       BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return "register";
    }
    // process
    return "redirect:/success";
}
</code></pre>

<p>Thymeleaf automatically exposes the <code>BindingResult</code> in the model under the name <code>BindingResult.{formObjectName}</code>. In this example, if the form object is called <code>user</code>, the <code>BindingResult</code> is accessible as <code>BindingResult.user</code>.</p>

<h2>Using <code>th:object</code> and <code>#fields</code>:</h2>
<pre><code>&lt;form th:object="${user}" action="#" method="post"&gt;
    &lt;input type="email" th:field="*{email}" /&gt;
    &lt;div th:if="${#fields.hasErrors('email')}" th:errors="*{email}"&gt;&lt;/div&gt;
&lt;/form&gt;
</code></pre>

<ul>
    <li><code>th:object="${user}"</code> tells Thymeleaf the current form object.</li>
    <li><code>#fields.hasErrors('email')</code> checks the <code>BindingResult</code> for the <code>user</code> object, equivalent to <code>BindingResult.user.hasFieldErrors('email')</code>.</li>
</ul>

<p>✅ So, <code>#fields</code> always refers to the <code>BindingResult</code> <strong>of the form object currently bound with <code>th:object</code></strong>.</p>

</body>
</html>

