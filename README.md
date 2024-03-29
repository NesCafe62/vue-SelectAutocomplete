# SelectAutocomplete

Select dropdown Vue component with autocomplete search ability.

## Demo

https://codepen.io/nescafe62/pen/mdyyQwK

## Usage

```js
import Vue from 'vue';
import SelectAutocomplete from 'src/SelectAutocomplete';

Vue.component('select-autocomplete', SelectAutocomplete);

new Vue({
    el: '#app',
    data: {
        options: [
            {value: 1, title: 'Option1'},
            {value: 2, title: 'Option2'},
        ]
    }
});
```

```html
<div id="app">
    <select-autocomplete
        :name="'field1'"
        :default-value="2"
        :placeholder="'-- Choose an option --'"
        :default-input-attribs="{tabindex: 1}"
        :default-options="options"
    ></select-autocomplete>
</div>
```
