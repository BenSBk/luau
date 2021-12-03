# Colon referencing

## Summary

Add a quality-of-life syntax addition where 'colon methods' can be referred to with a colon (without being called), evaluating to a wrapper function.

## Motivation

The currently existing colon syntaxes exist for two out of three common use cases of methods:

1. Defining methods ✅ (`function tab:fn(text) print(self.name..text) end`)
2. Calling methods ✅ (`tab:fn(" is my name")`)
3. Passing methods to higher order functions ❌

This feature would support passing colon methods to higher order functions. This would complete the colon syntax set and make Luau colon methods seamless to use across every common use case. It would improve code readability and reduce verbosity, rendering the two currently-used workarounds unnecessary.

For the common use case of pcalling `GlobalDataStore:GetAsync`, here are the possible approaches currently:

### Current approach 1

```lua
local isOk, result = pcall(dataStore.GetAsync, dataStore, key)
```

Here, we expose the truth/details behind colon methods, which is verbose and confusing. Ideally, a function defined/documented as a colon method should never be accessed with a `.`.

### Current approach 2

```lua
local isOk, result = pcall(function()
  return dataStore:GetAsync(key)
end)
```

Here, we use an anonymous function, which is verbose and adds an unnecessary line to the stack trace string which will make debugging slightly more difficult.

### Current approach 3

```lua
local function bind(o, m)
  return function(...)
    return o[m](o, ...)
  end
end

local getAsync = bind(dataStore, "GetAsync")

local isOk, result = pcall(getAsync, key)
```

With a bind function that works as shown (or similarly), boilerplate is added in both the function definition and the binding assignment to `getAsync`. When the function is passed to pcall, it may also be less immediately obvious that it's a binding to `dataStore:GetAsync` as the boilerplate code may be written near the top of the file while the pcall occurs lower down.

There is currently no approach that avoids the issues found in the current two and supports the third as-listed use of methods.

## Design

My proposed syntactical addition is the `a:b` expression, which evaluates to a function that, when called, returns `a:b(...)`, where `...` are the arguments passed to the function. `(a:b)()` would be equivalent to `a:b()`. To clarify:

```lua
local method = a:b

-- is similar to

local method = function(...)
  return a:b(...)
end
```

This would support the third use case as listed under the Motivation section: passing methods to higher order functions. For example:

```lua
local isOk, result = pcall(dataStore:GetAsync, key)

-- is the same as doing

local isOk, result = pcall(dataStore.GetAsync, dataStore, key)

-- but, in reality, this is happening:

local isOk, result = pcall(function(...)
  return dataStore:GetAsync(...)
end, key)
```

The expression `a:b` will error if:

1. Either `a` or `b` isn't an identifier,
2. `a[b]` errors (i.e. if `a` isn't indexable),
3. or `a[b]` evaluates to `nil`.

Even if `a[b]` evaluates to a non-function, non-functions may still be callable so there is nothing inherently wrong with the expression. If you use the syntax to get a wrapper for a non-function that isn't callable, you'd get an error when you call that wrapper, rather than when that wrapper is generated. The exception is `nil`, which can never be callable, so we just error immediately.

The resulting function is not be included in the stack trace string given by `debug.traceback` as well as the output upon errors. That is to say `(tab:fn)()`, where `fn` errors, produce the same stack trace string as `tab:fn()`.

The resulting function is cached, so `tab:fn == tab:fn --> true`.

`tab:fn` is not reported as equal to `tab.fn`, so `tab:fn == tab.fn --> false` and, obviously, `rawequal(tab:fn, tab.fn) --> false`.

## Drawbacks

Why should we *not* do this?

## Alternatives

What other designs have been considered? What is the impact of not doing this?
