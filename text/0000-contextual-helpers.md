- Start Date: 2018-12-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Contextual Helpers (a.k.a. "first-class helpers")

## Summary

We propose to extend the semantics of Handlebars helpers such that they can be
passed around as first-class values in templates.

For example:

```hbs
{{#let (helper "concat" "foo") as |foo|}}
  {{#let (helper foo "bar") as |foo-bar|}}
    {{foo-bar "baz"}}
  {{/let}}
{{/let}}
```

This template will print "foobarbaz".

## Motivation

[RFC #64](https://github.com/emberjs/rfcs/pull/64) introduced a feature known
as "contextual components", which allowed components to be passed around as
first-class values. While this is a somewhat advanced feature, it allowed addon
authors to encapsulate internal state and logic, which in turns, allowed them
to create easy-to-use and easy-to-understand DSL-like APIs that could benefit
users of all level.

For example, the original RFC used form controls as a motivating example.
Without contextual compoments, an addon that provides form-building components
might have to expose an API like this:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <SuperInput @model={{this.post}} @name="title" />
  <SuperTextarea @model={{this.post}} @name="body" />
  <SuperSubmit @form={{f}} />
</SuperForm>
```

As you can see, this is far from ideal for serveral reasons. First, to avoid
collision, the addon author had to prefix all the components. Second, the
`@model` argument has to be passed to all the controls that needs it. Finally,
in cases where the compoments need to communicate with each other (the form and
the submit button in the example), they would have to expose some internal state
(the `|f|` block param) that the user would have to manually thread through. Not
only does this make the API very verbose, it also breaks encapsulation.

Instead, the contextual components feature allows the addon author to expose an
API like this:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <f.Input @name="title" />
  <f.Textarea @name="body" />
  <f.Submit />
</SuperForm>
```

Behind the scene, the `<SuperForm>` compoment's template would look something
like this:

```hbs
<form ...>
  {{yield (hash
    Input=(component "super-input" form={{this}} model={{this.model}})
    Textarea=(component "super-textarea" form={{this}} model={{this.model}})
    Submit=(component "super-submit" form={{this}})
  )}}
</form>
```

Here, the `component` helper looked up the components by name (first argument)
and packaged up ("curried") any additional arguments (`form={{this}}`) into an
internal object (known as a "component definition" in the Glimmer VM). This
object can then be passed around like any other values and invoked at a later
time.

(Typically, a number of them are passed to the `hash` helper to "bundle" them
into a single object, but this is not required.)

While this is indeed a pretty advanced feature, the users of `SuperForm` do not
need to be aware of these implementation details in order to use the addon.
This had proved to be a very useful and powerful feature and enabled a number
of popular addons, such as **@kategengler send halp**.

The original RFC left an important question unanswered – [should this feature
be available for helpers, too?](https://github.com/emberjs/rfcs/blob/master/text/0064-contextual-component-lookup.md#unresolved-questions)

In this RFC, we argue – yes, this feature will be just as useful for helpers as
well as modifiers.

For example, the `SuperForm` addon API can be expanded to include some extra
helpers and modifiers, like so:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <f.Input @name="title" />

  {{!-- f.is-valid and f.error-for are contextual helpers --}}
  {{#unless (f.is-valid "title")}}
    <div class="error">This field {{f.error-for "title"}}</div>
  {{/unless}}

  {{!-- f.auto-resize is a contextual modifier --}}
  <f.Textarea @name="body" {{f.auto-resize maxHeight="500"}} />

  <F.Submit />
</SuperForm>
```

For reference, the `<SuperForm>` compoment's template would look something like
this:

```hbs
<form ...>
  {{yield (hash
    is-valid=(helper "super-is-valid" form=this model=this.model)
    error-for=(helper "super-error-for" form=this model=this.model)
    auto-resize=(modifier "super-auto-resize")
    ...
  )}}
</form>
```

This RFC proposes a complete design for enabling this capability.

## Detailed design

### The `helper` and `modifier` helpers

This RFC introduce two new helpers named `helper` and `modifier`, which work
similarly to the `component` helper:

* When passed a string (e.g. `(helper "foo")`) as the first argument, it will
  produce an opaque, internal helper definition" object that can be passed
  around and used to invoke the corresponding helper (the `foo` helper in this
  case) in Handlebars.

* When passed an "helper definition" as the first argument, it will produce a
  functionally equivialent "helper definition" object.

* In either case, any additional positional and/or named arguments will be
  stored ("curried") inside the "helper definition" object, such that, when
  eventually invoked, these arguments will be passed along to the referenced
  helper.

Some additional details:

* The string will be used to resolve a helper with the same name. If the string
  does not correspond to a valid helper, it will result in a runtime error.
  However, the timing of this lookup is unspecified. `(helper "not-a-helper")`
  may result in an immediate error, or it may happen when it is later passed
  into the `helper` helper a second time, or when it is invoked. If it is never
  invoked, the error may not be reported at all. This timing may change between
  versions, and should not be relied upon.

* Positional arguments are "curried" the same way as the `component` helper
  (which matches the behavior of `Function.prototype.bind`).

  ```hbs
  {{#let (helper "concat") as |my-concat|}}
    {{my-concat "foo" "bar" "baz"}} {{!-- "foobarbaz" --}}

    {{#let (helper concat "foo") as |foo|}}
      {{foo "bar" "baz"}} {{!-- "foobarbaz" --}}

      {{#let (helper foo "bar") as |foo-bar|}}
        {{foo-bar "baz"}} {{!-- "foobarbaz" --}}
      {{/let}}

    {{/let}}

  {{/let}}
  ```

* Named arguments are "curried" the same way as the `component` helper (which
  is "last-write-wins", matching the behavior of `Object.assign`).

  ```hbs
  {{#let (helper "hash") as |my-hash|}}
    {{my-hash value="foo"}} {{!-- hash with value="foo" --}}

    {{#let (helper my-hash value="foo") as |foo|}}
      {{foo value="bar"}} {{!-- hash with value="bar" --}}

      {{#let (helper foo value="bar") as |bar|}}
        {{bar value="baz"}} {{!-- hash with value="baz" --}}
      {{/let}}

    {{/let}}

  {{/let}}
  ```

* When a "helper definition" is passed into JavaScipt, the resulting value is
  left unspecified (which is why it is described as "opaque"). In particular,
  it is _not_ guarenteed that it will be an invokable JavaScript function. The
  only guarentee provided is that, when passed back into Handlebars, it will be
  functionally equivilant to the original "helper definition". Hanging onto a
  "helper definition" object in JavaScript may result in unexpected memory
  leaks, as these objects may "close over" arbitrary template states to allow
  currying.

### Invoking contextual helpers

For the most part, invoking a contextual helper is no different from invoking
any other helpers:

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- content position --}}
  {{foo-bar "baz"}}

  {{!-- not necessary, but works --}}
  {{helper foo-bar "baz"}}

  {{!-- attribute position --}}
  <div class={{foo-bar "baz"}}>...</div>

  {{!-- curly invokcation, argument position --}}
  {{MyComponent value=(foo-bar "baz")}}

  {{!-- angle bracket invokation, argument position --}}
  <MyComponent @value={{foo-bar "baz"}} />

  {{!-- sub-expression positions --}}
  {{yield (hash foo-bar=foo-bar)}}

  {{#if (eq (foo-bar "baz") "foobarbaz")}}...{{/if}}

  {{!-- runtime error: not a component --}}
  <foo-bar />

  {{!-- runtime error: not a modifier --}}
  <div {{foo-bar "baz"}}>
{{/let}}
```

### Ambigious invocations (invoking without arguments)

When invoking a contextual helper without arguments, the invokation becomes
ambigious. Consider this:

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- content position --}}

  {{!-- does this render "foobar", or something like "[object Object]" ? --}}
  {{foo-bar}}

  {{!-- attribute position --}}

  {{!-- class="foobar", or something like class="[object Object]" ? --}}
  <div class={{foo-bar}}>...</div>

  {{!-- curly invokcation, argument position --}}

  {{!-- is this.value the string "foobar", or the "helper definition" ? --}}
  {{MyComponent value=foo-bar}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- is @value the string "foobar", or the "helper definition" ? --}}
  <MyComponent @value={{foo-bar}} />

  {{!-- sub-expression positions --}}

  {{!-- are we yielding the helper, or the string "foobar" ? --}}
  {{yield (hash foo-bar=foo-bar)}}

  {{!-- is this true or false? --}}
  {{#if (eq foo-bar "foobar")}}...{{/if}}
{{/let}}
```

In these cases, these are ambigious between passing the _result_ of the helper
(first invoking the helper, then pass along the result), or passing the helper
itself (the "helper definition" object, so that it can be invoked or "curried"
again on the receiving side).

Since this RFC proposes to treat helpers and modifiers as first-class values,
they should generally be passed through as _values_. This is particularly
important in arguments and sub-expression positions. To invoke the helper, the
explicit `(some-helper)` syntax can be used instead:

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- curly invokcation, argument position --}}

  {{!-- the component will receive the "helper definition" --}}
  {{MyComponent value=foo-bar}}

  {{!-- the component will receive the string "foobar" --}}
  {{MyComponent value=(foo-bar)}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- @value will be the "helper definition" --}}
  <MyComponent @value={{foo-bar}} />

  {{!-- @value will be the string "foobar" --}}
  <MyComponent @value={{(foo-bar)}} />

  {{!-- sub-expression positions --}}

  {{!-- yielding the helper --}}
  {{yield (hash foo-bar=foo-bar)}}

  {{!-- yielding the string "foobar" --}}
  {{yield (hash foo-bar=(foo-bar))}}

  {{!-- false --}}
  {{#if (eq foo-bar "foobar")}}...{{/if}}

  {{!-- true --}}
  {{#if (eq (foo-bar) "foobar")}}...{{/if}}
{{/let}}
```

However, in the case of content and attribute positions, it would be overly
pendantic to insist on the `{{(some-helper)}}` syntax, as the alternative of
printing `[object Object]` to the DOM is almost certainly not what the
developer had in mind. Therefore, we propose to allow contextual helpers to be
auto-invoked in these positions.

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- content position --}}

  {{!-- "foobar" --}}
  {{foo-bar}}

  {{!-- not necessary, but works --}}
  {{(foo-bar)}}

  {{!-- attribute position --}}

  {{!-- class="foobar" --}}
  <div class={{foo-bar}}>...</div>

  {{!-- class="foobar" --}}
  <div class={{(foo-bar)}}>...</div>
{{/let}}
```

It should also be noted that modifiers doe not have the same problem, since
there are no other possible meaning in that position:

```hbs
{{#let (modifier "foo-bar") as |foo-bar|}}
  {{!-- modifier position: not ambigious --}}
  <div {{foo-bar}} />

  {{!-- not necessary, but works --}}
  <div {{(foo-bar)}} />

  {{!-- undefined behavior: runtime error or [object Object] ? --}}
  {{foo-bar}}

  {{!-- undefined behavior: runtime error or [object Object] ? --}}
  <div class={{foo-bar}} />
{{/let}}
```

### Deprecation

In today's Ember, "global helpers" (as opposed to "contextual helpers") does
not always follow the rules laid out above. In particular, they do not behave
the same way in arguments and subexpression positions:

```hbs
{{#let (helper "concat") as |my-concat|}}
  {{!-- curly invokcation, argument position --}}

  {{!-- this.value is the helper --}}
  {{MyComponent value=my-concat}}

  {{!-- this.value is the undefined --}}
  {{MyComponent value=concat}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- @value is the helper --}}
  <MyComponent @value={{my-concat}} />

  {{!-- @value is an empty string (invoking concat with no arguments) --}}
  <MyComponent @value={{concat}} />

  {{!-- sub-expression positions --}}

  {{!-- yielding the helper --}}
  {{yield (hash value=my-concat)}}

  {{!-- yielding undefined --}}
  {{yield (hash value=concat)}}

  {{!-- false: it compares the helper with undefined --}}
  {{#if (eq my-concat concat)}}...{{/if}}
{{/let}}
```

This illustrates the problem: "global helpers" are not modelled as first-class
values today, they exists in a different "namespace" distinct from the "local
variables" namespace.

While this is not so different from other programming languages like Java and
Ruby which also do not treat functions (methods) as first-class values, it is
distinctly different from JavaScript which does. For example, `alert("hello")`
is the same as `let a = alert; a("hello");`, whereas in Java and Ruby (and
today's Handlebars), the moral equivilant of the latter would fail with `alert`
being an undefined reference.

The goal of this RFC is to iterate the Ember Handlebars programming model
towards a world closer to JavaScript's, where global names exists in the same
namespace as local names. We propose to deprecate all cases where global names
("global helpers", "global modifiers" and "global components") can be observed
to behave differently.

Specifically:

```hbs
{{#let (helper "concat") as |my-concat|}}
  {{!-- curly invokcation, argument position --}}

  {{!-- deprecation: use `this.concat` instead --}}
  {{MyComponent value=concat}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- deprecation: use `{{(concat)}}` instead --}}
  <MyComponent @value={{concat}} />

  {{!-- sub-expression positions --}}

  {{!-- deprecation: use `this.concat` instead --}}
  {{yield (hash value=concat)}}

  {{!-- depreaction: use `this.concat` instead --}}
  {{#if (eq my-concat concat)}}...{{/if}}
{{/let}}
```

Overall, we expect the effect of this deprecation to be quite minimal. For the
cases that triggers a property lookup today, they are already covered in the
[Property Lookup Fallback Deprecation RFC](https://github.com/emberjs/rfcs/blob/master/text/0308-deprecate-property-lookup-fallback.md),
plus it would already be quite confusing to name an instance variable after a
global helper. For the cases where the "global helper" is implicitly invoked
without arguments, since helpers are supposed to be pure computations, a helper
that doesn't accept any arguments have very limited utility thus should also be
quite rare.

### Local helpers (and modifiers)

A nice fallout of this plan is that developers will be able to define helpers
specific to a component locally (i.e. in the same JavaScript file):

```js
// app/components/date-picker.js

import Component from '@ember/component';
import { helper } from '@ember/component/helper';

export default Component.extend({
  date: null, // passed in

  "format-date": helper(function(params) {
    /* ... */
  })
});
```

```hbs
{{!-- app/templates/components/date-picker.hbs --}}

{{input value=(this.format-date this.date)}}
```

In additional to encapsulation and namespacing, this will also enable even more
advanced stateful use cases:

```js
// app/components/filtered-each.js

import Component from '@ember/component';
import { helper } from '@ember/component/helper';
import { computed } from '@ember/object';

export default Component.extend({
  list: null, // passed in
  callback: null, // passed in

  filter: computed('callback', function() {
    return helper(params => this.callback(params[0]));
  });
});
```

```hbs
{{!-- app/templates/components/filtered-each.hbs --}}

{{#each this.list as |item|}}
  {{#if (this.filter item)}}
    {{yield item}}
  {{/if}}
{{/each}}
```

### Rationalization

We propose to rationalize the existing and proposed semantics into a coherent
model at a deeper level. This knowledge is necessary for regular day-to-day
use, but could be helpful for guiding the implementation as well as design of
future feature proposals.

Today, the Handlebars mustache

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
