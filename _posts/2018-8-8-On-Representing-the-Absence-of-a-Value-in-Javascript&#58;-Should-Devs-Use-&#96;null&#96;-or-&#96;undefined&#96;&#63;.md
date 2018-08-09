tl;dr: use `null`.

Many things in a JS runtime environment can turn out to be `undefined`. According to ES specs, "Any variable that has not been assigned a value has the value undefined" (6.1.1: "The Undefined Type"). It strikes me as confusing to tack onto this, "any variable that has been assigned a value can also have the value undefined, to signify the absence of any value."

By contrast, according to MDN, "The value `null` represents the intentional [developer-generated] absence of any object value."

Conclusion: best practice/convention: use `null`.
