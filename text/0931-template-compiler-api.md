---
stage: accepted
start-date: 2023-05-31 
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/931
project-link:
suite:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# JS Representation of Template Tag

## Summary

Formalize a Javascript-spec-compliant representation of template tag.

## Motivation

The goal of this RFC is to improve the plain-Javascript representation of the Template Tag (aka "first-class component templates") feature in order to:

- reduce the number and complexity of API calls required to represent a component
- efficiently coordinate between different layers of build-time and static-analysis tooling that need to use preprocessing to understand our GJS syntax extensions.
- avoid exposing a "bare template" as a user-visible value
- provide a declarative way to opt-in to run-time (as opposed to build-time) template compilation.
- enable support for class private fields in components

As an illustrative example, currently this template tag expression:

```js
let x = <template>Hello {{ message }}</template>;
```

Has the plain javascript representation:

```js
import { precompileTemplate } from "@ember/component";
import templateOnlyComponent from "@ember/component/template-only";
import { setComponentTemplate } from "@ember/component";
let x = setComponentTemplate(
  precompileTemplate("Hello {{message}}", {
    strictMode: true,
    scope: () => ({ message }),
  }),
  templateOnlyComponent()
);
```

This RFC proposes simplifying the above case to:

```js
import { template } from "@ember/template-compiler";
let x = template("Hello {{message}}", {
  scope: () => ({ message }),
});
```

As a second illustrative example, currently this class member template:

```js
class Example extends Component {
  <template>Hello {{message}}</template>
}
```

Has the plain javascript representation:

```js
import { precompileTemplate } from "@ember/component";
import { setComponentTemplate } from "@ember/component";
class Example extends Component {}
setComponentTemplate(
  precompileTemplate("Hello {{message}}", {
    strictMode: true,
    scope: () => ({ message }),
  }),
  Example
);
```

This RFC proposes simplifying the above case to:

```js
import { template } from "@ember/template-compiler";
class Example extends Component {
  static {
    template(
      "Hello {{message}}",
      {
        component: this,
        scope: () => ({ message }),
      },
    );
  }
}
```

## Detailed design

This RFC introduces two new importable APIs:

```js
// The ahead-of-time template compiler:
import { template } from "@ember/template-compiler";

// The runtime template compiler:
import { template } from "@ember/template-compiler/runtime";
```

They are intended to be drop-in replacements for each other _except_ for the differences summarized in this table:

|               | Template Contents      | Scope Param                                  | Syntax Errors            | Payload          |
| ------------- | ---------------------- | -------------------------------------------- | ------------------------ | ---------------- |
| Ahead-of-Time | Restricted to literals | Restricted to a few explicitly-allowed forms | Stops your build         | Smaller & Faster |
| Runtime       | Unrestricted           | Unrestricted                                 | Can by caught at runtime | Larger & Slower  |

By putting these two implementations in different importable modules, the problem of "how do you opt-in to including the template compiler in your app?" goes away. If you import it, you will have it, if you don't, you won't.

The remainder of this design only uses examples with the ahead-of-time template compiler, because everything about the runtime template compiler's API is the same.

### Scope Access

To give templates access to the relevant Javascript scope, we offer **two different forms** for two different use cases. A critical feature of this design is that *both forms* have valid Javascript syntax **and** semantics. That means they can actually run when you want them to, with no further processing. And they are fully understandable by all spec-compliant Javascript tools. This is in contrast with intermediate forms like `[__GLIMMER_TEMPLATE("<HelloWorld />")]` in the current ember-template-imports implementation or the proposed:

```js
template`<HelloWorld />`
```

from [RFC 813](https://github.com/emberjs/rfcs/pull/813), which both lack any mechanism to access the surrounding scope, and therefore need "magic" beyond Javascript to make them run.

#### Explicit Form

The "Explicit Form" makes all data flow statically visible. It's the appropriate form to publish to NPM. To produce Explicit Form, build tools need to do a full parse of the template and a full lexical analysis of Javascript and Handlebars scopes.

Examples of Explicit Form:

```js
import { template } from '@ember/template-compiler';

// when nothing is needed from scope, no scope params are required:
const Headline = template("<h1>{{yield}}</h1>");

// local variable access works the same as in current precompileTemplate
const Section = template(
  "<Headline>{{@title}}</Headline>", 
  { 
    scope: () => ({ Headline}) 
  }
);

// in class member position, we can also put private fields in scope
class extends Component {
  static {
    template(
      "<Section @title={{this.title}} @subhead={{this.#secret}} />", 
      {
        component: this,
        scope: (instance) => ({ 
          Section, 
          "#secret": instance.#secret
        }),
      },
    )
  }
}
```

> This RFC is focused on making sure the scope accessors can do everything javascript can do, which is why we're including private fields. However, additional work beyond this RFC is required to make the template compiler correctly parse expressions like `{{this.#secret}}`. 

#### Implicit Form

The "Implicit Form" is cheaper and easier to produce because it doesn't need to do any lexical analysis and doesn't need to parse the handlebars at all.

The downside is that data flow is not all statically visible, because it relies on `eval`.

Implicit Form has two key use cases:
 - as an intermediate format between a preprocessor stage (which can eliminate all GJS special syntax and semantics) and the rest of a standard Javascript toolchain.
 - as the implementation format in sandbox-like environments where dynamic code execution is the whole point.

Examples of Implicit Form:

```js
import { template } from '@ember/template-compiler';

// Notice that all of these have the exact same
// `params` argument. It's always the same. 
// That's why it's easy to produce.

const Headline = template(
  "<h1>{{yield}}</h1>",
  {
    eval() { return eval(arguments[0]); }
  }
);

const Section = template(
  "<Headline>{{@title}}</Headline>", 
  { 
    eval() { return eval(arguments[0]); }
  }
);

class extends Component {
  static {
    template(
      "<Section @title={{this.title}} @subhead={{this.#secret}} />", 
      {
        component: this,
        // this handles private fields just fine,
        // see Appendix A.
        eval() { return eval(arguments[0]); }
      },
    )
  }
}
```

> _eval seems bad, what about Content Security Policy?_<br>
> Typical apps never needs to actually _run_ the eval. This is a communication format between layers of build tooling. You _can_ run it, if you're making something like an interactive development sandbox. But that is a case that already requires `eval` anyway.

> _Why `arguments[0]` instead of an explicit argument?_<br>
> If we picked a local name to use for the argument, we would shadow that name in the surrounding scope. Whereas `arguments` is already a keyword that exists for this purpose, and can never collide with other local bindings.


### Type Signature

```ts
import { ComponentLike } from '@glint/template';

declare function template<S>(
  templateContent: string,
  params?: Params<never>
): ComponentLike<S>;

declare function template<C extends ComponentLike<any>>(
  templateContent: string,
  params: Params<C>
): C;

interface Params<ComponentClass> {
  component?: ComponentClass;
  strict?: boolean;
  moduleName?: string;
  eval?: () => Record<string, unknown>;
  scope?: (
    instance: ComponentClass extends ComponentLike<any>
      ? InstanceType<ComponentClass>
      : never
  ) => Record<string, unknown>;
}

```

### Strict defaults to true

Unlike `precompileTemplate`, our `strict` param defaults to true instead of false if it's not provided. This is aligned with the expectation that our main programming model is moving everyone toward handlebars strict mode by default.

This also addresses the naming confusing between earlier RFCs (which used "strict") and the implementations in the ecosystem (which used "strictMode").

### Always Returns a Component

A key difference between `precompileTemplate` and our new `template` is that its return value is always a _component_, never a "bare template". In this sense, the implementation combines the jobs of `precompileTemplate` and `setComponentTemplate`.

Bare templates are a historical concept that we'd like to move away from, in order to have fewer things to teach.

When the `component` argument is passed, the return value is that backing class, with the template associated, just like `setComponentTemplate`. When the `component` argument is not provided, it creates and returns a new template-only component.

> _Aren't route templates "bare templates"? What about them?<br>_
> Yes, this RFC deliberately doesn't say anything about route templates. We expect a future routing RFC to use components to express what today's route templates express. This RFC also doesn't deprecate `precompileTemplate` yet -- although that will clearly be a good goal _after_ a new routing design addresses route templates.

### Syntactic Restrictions

The runtime template compiler has no syntactic restrictions.

The ahead-of-time template compiler has syntactic restrictions on `templateContents`, `params.scope`, and `params.eval`.

`templateContents` must be one of:

- a string literal
- a template literal with no expressions

If provided, `params.scope` must be:

- an arrow function expression or function expression
  - that accepts zero or one arguments
  - with body containing either
    - an expression
    - or a block statement that contains exactly one return statement
  - where the return value is an object literal
    - whose properties are all non-computed
    - whose values are all either
      - identifiers
      - or private field member expressions on our argument identifier

If provided, `params.eval` must be:
 - an object method
 - whose body contains exactly one return statment.
 - where the return value must be exactly `eval(arguments[0])`.


### Older things that are intentionally dropped

`precompileTemplate` and its predecessors like `import { hbs } from 'ember-cli-htmlbars'` accepted some additional params that we are choosing not to keep.

 - `insertRuntimeErrors`: instead you should use `@ember/template-compiler/runtime` and catch the exception thrown by `template()`.

## How we teach this

Mentioning these APIs is appropriate when introducing the template tag feature in the guides. We can explain template tag as a convenience over these lower-level APIs and show side-by-side how a given template tag "really means" a call to `template()`. These examples should probably use the Explicit Form.

We can also mention that fully runtime template compilation is possible using `@ember/template-compiler/runtime` and show this example in Implicit Form, since that pairs well with a dynamic, REPL-like environment.

## Drawbacks



## Alternatives

This RFC builds off the proposal in
https://github.com/emberjs/rfcs/pull/813.

The main difference is that RFC 813 offered a form:

```
template`<Foo />`
```

that converts `<template>` into valid JS syntax _without_ working JS semantics.

### Type Strictness Alternative

The typescript types as written above don't prevent you from using `scope` and
`eval` simultaneously. We could use more complicated types to enforce that. See [example](https://www.typescriptlang.org/play?#code/PTAEFpM0GUEsDmA7AhgFwK4CcCmoAmOAZnEnGnAPZIRTiggBQjcAtgA6VZqgDkAAggA2pNMBxIAbnCzVWEtOBysARjizghlSgGccvANwsOXHgG9QAYUomkCgDJwA1ngC+oIrNZ9BIpGLRldiF0fSM2Tm4rG047fw8vH2E2eSxgAGMY6gVDZkJ0kNwPDCR0impQQI4QwIAeGAA+AApGUEqgmpxrfwUALlAdNCxSBAAaVtB2FCwUVh0Afn6YIbgygFUyalqAUQAPYNXyAAVp2Z0AMS4AFQ7QgHkkIQBPUAAfUABJasO0E5m5hqMACU-Wstgczhw9QaRnyhTwRBKZSoNCqwVCtUsoBwu0CSHwOmi4P8jhctRQSCeDWaEzRnW6eLQ-UGwyQYwmKhQ6ScI0sIR0OlB4zaU3+CyWK3WmyQO32InSx1Oc0uWD5KAFACEuS58JiGm9Pt8Fb8lTpASCrEYWD0sEQuXgtXo-mdQGYJizVmhFqAVNohDgKUY2qxKPgMP6AHKzHDej1soyuZiidR29J4PYHY3O5XXW6BB7PbG4iQE0COnDZwluto6TLsGP9JpA0AAXn1ACUcJksLq42NQCUnEhKAB3JAwxiJ62BW320AZ+WKsUqtWa7U4XVYnF40tg2IQskUqmAtrbkuE8uV13uusN0BNUiDClp-ofJBP0o4K5Pet65tt0BO27XsVjZUYByQIdR3HBMkxtVM8C+TMlxdM98QvdUK1Na9T0kFAhEbf99UHYcx1gxg0B-PANhRABpHAnh0Worn1FtQCuIsd0JDj5lAFwnkoIh2NAfo7EkdQjEo+tYElNAaOoAAJHAhHrLBmPAq4AEEhCEVjhLQ0sjwmXiOIAMlAP4KHw2ogK4XU9gKDBCFqeSkHoxjmO03TwP4wT2IacCSkIEg7HwakJlEnBxKwSSqJk4YpRRZi9OWBK5OlJSVPUdT-KtEBaCgeddlmYIcEJOh6CYRhMnfHgtHSfDWz4EcuCEfBckYRFSnKGgcQKFA2CaQUBlAhBm2rUBcEwLAaAAAwAdQAC0oFAAEJQAAEjMHRXFm8j8sAGXJGHy-g0B0JR9i7RR1FkLBaFAEc8AapBhx4KYBR9Sg0EWgZb1ACl8GxPChAAOjBii8xwJpZtqdgGkyrRwLMMx6vw1xXFW2pgDh2akYmKL8KbHC2kmnBpt64GmmmBAMHkfwdAAbQABgAXSBINQFcYVfsoetCNbfUmgsVGhE5oFxlcdnmHywBQcgh6pQmh2H4eUxHXRRygGqEdHMexhpceJ2teZwfmAKF0ARbFiWpeOsA5bpRWYbhhHKCRjWtZ1rGcbx3DCfGiY2im7AKcJ6naYURnWaltoucnG3Zeq-lCT2Er-QARk488iX3fxagsTSsAQYaLFQeRmVGgx-oQE3QCQWm1Du9xXH1Cbq54ABiQhaybcvWQQYnA7J4PQFmrbvrgHQQepyfS5wdwJ82sxx8n6ep+r9wngDLBCUoNqQb2iYpxrNB0FWAf2gVwIlbh5G+pCNhKkWieQc7sr0nRr39fA5efZJmtb0bI+E+n4iL3gmn-U8ux+psG5hAvgr9ay8H6EA58OAX5d3SLAkmkssGSw5omKcCd+ofRTtUHAAAmTO6Fs7ZFzvnQuxda7Rl7iMSuKBq6iXruoTmnMW4TDbqABB6Qe4jT7ufIOM0R5jyfivBhINZ7z0JNI5+q92Fz1AJvaYO894HxjswY+p90jnwdlfJ2DRb5QPvt4Ze6C34fz1gbH+59fZCCJuAiBEiQ6uLDnTM6zM2Ycz-rHIJ0dOaTmlmAA6909B4FengCos0CZCFmj6ZSo5E7qmTsVMhABmKhu4shxDQHnUABci79BLsw0RrCq41zrqobhTc+FtAEUIkRfZxFD0kaPJeMip5yIUaABeyjZFFzXuozR29QC73wPvfB+iRqGOMZDa+5izB3wGtYvpQj7He0fhPX+f8jZ83vCgkBAswEBwgRsmBVy-68CEUgoZtVUG2NrFgmO4s7kMDAKdc6OJ6xlCUFgW691HqgGeq9SYmTPrfR5tJAGQN8JgxBt8pJbjvmD3Jki7xhdw7038aEoJuDQkEMYEAA) courtesy of @chriskrycho. I didn't go with that in the RFC because it's harder to read and explain, and I don't think this low-level API present much practical risk that people will be accidentally including both together. 


## Unresolved questions

### Appendix A: Field Access Patterns

This is a fully-working example that can run in a browser. It uses a toy rendering engine just to illustrate how scope access is working.

```html
<script type="module">
  let templates = new WeakMap();

  function template(content, params) {
    templates.set(params.component, { content, params });
  }

  function render(instance) {
    let { content, params } = templates.get(instance.constructor);
    return content.replace(/\{\{([^}]*)\}\}/g, (m, g) =>
      get(instance, params, g)
    );
  }

  function get(instance, params, path) {
    if (params.eval) {
      if (path.startsWith("this.")) {
        return params.eval("arguments[1]." + path.slice(5), instance);
      }
      return params.eval(path);
    } else {
      if (path.startsWith("this.#")) {
        return params.scope(instance)[path.slice(5)];
      } else if (path.startsWith("this.")) {
        return instance[path.slice(5)];
      } else {
        return params.scope(instance)[path];
      }
    }
  }

  let local = "I'm a local variable";
  class ImplicitExample {
    publicField = "I'm a public field";
    #privateField = "I'm a private field";
    static {
      template(
        `DymamicExample
        publicField={{this.publicField}}, privateField={{this.#privateField}}, local={{local}}
        `,
        {
          component: this,
          eval() {
            return eval(arguments[0]);
          },
        }
      );
    }
  }

  class ExplicitExample {
    publicField = "I'm a public field";
    #privateField = "I'm a private field";
    static {
      template(
        `StaticExample:
        publicField={{this.publicField}}, privateField={{this.#privateField}}, local={{local}}
        `,
        {
          component: this,
          scope: (instance) => ({
            local,
            "#privateField": instance.#privateField,
          }),
        }
      );
    }
  }

  console.log(render(new ImplicitExample()));
  console.log(render(new ExplicitExample()));
</script>
```
