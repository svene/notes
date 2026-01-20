# AlpineJS Tips and Tricks

## x-bind

````html
<!-- bind attribute 'class': -->
<input x-bind:class="{'dirty': dirty}">

<!-- bind boolean attribute 'disabled': -->
<button x-bind:disabled="!dirty">Reset</button>

<!-- bind multiple classes in attribute 'class' : -->
<h1 x-bind:class="{'pico-color-pumpkin': dirty, 'dirty-title': dirty}" style="align-content: center"></h1>
````
