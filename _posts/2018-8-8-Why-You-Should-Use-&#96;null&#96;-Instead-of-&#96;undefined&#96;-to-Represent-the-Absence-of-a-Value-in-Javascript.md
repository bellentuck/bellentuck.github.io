Many entities in a JS runtime environment can turn out to be `undefined`. [According to ES specs](http://www.ecma-international.org/ecma-262/6.0/#sec-ecmascript-language-types-undefined-type), "Any variable that has not been assigned a value has the value undefined." It strikes me as confusing to conceptually tack onto this, "any variable that has been assigned a value can *also* have the value undefined, to signify the absence of any value."

By contrast, [according to MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/null), "The value `null` represents the intentional [i.e., developer-generated] absence of any object value."

Conclusion: best practice/convention: use `null`.
