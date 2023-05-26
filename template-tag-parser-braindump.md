# 🧠 `<template>` Tag Parser Braindump

This is a braindump on the conversations had in the 23/05/2023 Embroider Office Hours around the extraction of the template tag preprocessor and babel-plugin as per [ember-template-imports/issues#143](https://github.com/ember-template-imports/ember-template-imports/issues/143)

## What’s the reason for the extraction?

ember-template-imports is an Ember addon intended to demonstrate potential approaches to using template imports, also known as “single file components” or “first class component templates”, in Ember apps. The two approaches that the addon supported were:

**Tagged template literals**, eg
```js
import { hbs } from '@glimmer/component';

const Greeting = hbs`Hello!`;
```

**Template tags**, eg
```js
const Greeting = <template>Hello!</template>
```

For the purpose of this document, we’re only going to focus on the latter option as that has been decided to be the intended approach for the future and it is now referred to as a `<template>` tag.

The `<template>` tag is not valid JS so ember-template-imports implements code to parse .gjs/.gts files and to convert the `<template>` tags into valid JS.

Over time, various other addons have begun to depend on ember-template-imports as they need a way to parse templates that include `<template>` tags. [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint) and [eslint-plugin-ember](https://github.com/ember-cli/eslint-plugin-ember) are two examples. However, these addons don’t need the entire functionality of ember-template-imports, just the ability to parse a template so it becomes awkwards and brittle that these addons start depending on an addon that was intended to demonstrate two potential solutions to a problem.

Add to that that ember-template-imports is a V1 addon which means that even if the consuming addons were V2 addons, they’d still be depending on a V1 addon which is problematic from an Embroider perspective.

So, with this, it was decided that the parsing and processing logic of ember-template-imports be extracted in to standard npm packages that could be depended upon by ember-template-imports itself and any other addons that needed the ability to parse templates.

## What are the issues with the extraction?
It became apparent very quickly that there is a bunch of interconnectedness between files in ember-template-imports and outside addons depending on various bits of code in the addon. This image paints some of the picture:

![image](https://github.com/achambers/ember-template-imports/assets/416724/b9bdf3a8-fee1-4f1f-a97f-942b06497129)

An example of where this becomes problematic is ember-template-lint. As mentioned above, the tagged template literal approach (hbs``) is not the supported approach for first class component templates and so the ember-template-imports is not going to support it. However, ember-template-lint relies on the template parsing code to detect tagged template literals. This is the wrong approach for ember-template-lint as there are other packages that should be used to detect this. The dependencies here are wrong and brittle.

Not to mention, it’s unnecessary for Ember addons to need to depend on the ember-template-imports addon - which is intended to add `<template>` tag support to an Ember app - when they only want the template parsing functionality. As ember-template-imports is a V1 add-on this means that even if you have a V2 add-on in your Embroider app, it will still be pulling a V1 add-on into the build.

The greater issue here, however, is that the implementation of the gts parser is not satisfactory. In fact, there are use cases that it doesn’t support, contains bugs and is not the correct way to parse a template. Remember, it was a proof of concept addon to demonstrate potential solutions. It wasn’t really created for production usage.

Before we look at what the new plan of attack is, let’s better understand what is needed in order to support `<template>` tags, why ember-template-imports isn’t satisfactory and why this matters to Ember and Embroider.

### What does ember-template-imports actually do?
When we run a build across our Javascript we run it through Babel which converts the JS in to an AST and then babel plugins run across the AST or transform it in a multitude of ways (think converting arrow functions into standard function calls (forget the fact arrow functions are supported by modern browsers now - this was a use case once)).

This Babel process consists of two main phases - parsing the JS which is turning the valid JS into an AST, and transforming the JS which is modifying the AST to output the destination JS.

Taking the `<template>` tag as an example, we essentially want to convert this code:

```js
// as an expression
const GreetingExpression = <template>Hello!</template>

// as a class property
class GreetingClass {
  <template>Hello!</template>
}
```

into this:

```js
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

// as an expression
const GreetingExpression = setComponentTemplate(
  precompileTemplate(`Hello!`, { /*scope*/ }),
  templateOnlyComponent('greeting.js', 'Hello!')
);

// as a class property
class GreetingClass {}
setComponentTemplate(precompileTemplate(`Hello!`, { /*scope*/ }), GreetingClass);
```

which are the current Ember APIs to compile templates.

The problem though is that we can’t pass a .gjs file to Babel because the `<template>` tag is not valid JS and Babel does not understand how to convert it to an AST.

Therefore, we need to do some pre-compilation of the .gjs file before handing it to Babel, turning the `<template>` tag to some other intermediate format that is valid JS. Therefore, the precompiler in ember-template-imports takes the source .gjs content and converts it to this:

```js
// as an expression
const GreetingExpression = [__GLIMMER_TEMPLATE(`Hello!`, { /*scope*/ })];

// as a class property
class GreetingClass {
  [__GLIMMER_TEMPLATE(`Hello!`, { /*scope*/ })]
}
```

This is now valid JS which can be parsed by Babel which will then be passed in to a babel-plugin, provided by ember-template-imports, that converts from the intermediate JS format to the final destination JS.

### What’s problematic about the current implementation?

There are two main problems with the current pre-processor as it is today. The first is that it just uses a regex to determine the `<template>` tags which is brittle and also means certain use cases are tricky, or impossible to handle. The second is that the intermediate format is syntactically valid JS but it’s not semantically correct. It’s just a format that happens to be valid JS and serves the purpose of a placeholder to hook into in the Babel transform phase. Let’s dive into each of those items.

#### It uses regex to detect `<template>` tags
As we’ve mentioned, Babel can’t parse the .gts files natively as it doesn’t understand `<template>` tags. Therefore, ember-template-imports uses a regex to detect, inside the template string, instances of the `<template>` tag in order to do the precompile before Babel parses the JS.

This approach becomes problematic because the “parser” doesn’t really understand the JS. It doesn’t understand the difference between a `<template>` tag used in an expression context, eg

```js
// as an expression
const GreetingExpression = <template>Hello!</template>
```
vs a `<template>` tag used in a class property context, eg

```js
// as a class property
class GreetingClass {
  <template>Hello!</template>
}
```

and therefore does not understand the scopes involved with them. The former needs to be handled slightly differently than the latter.

Not to mention, use cases like this are not possible to detect:

```js
const myregex = /<template>/;
<template>Hello!</template>
```

where `<template>` is actually a part of a regex and not intended to be the `<template>` tag. Because this parser is a regex and not an actual AST parser, it is unable to understand the context.

Parsers like Babel, allow us to create plugins that transform the AST in to different shapes, but they don’t support the ability to plugin to the actual parsing logic that generates the AST. This is the reason that the current approach uses the regex and intermediate format.

---

TODO: Put something in here about scope and maybe an example. Not yet quite clear on this.

---

#### The intermediate format is problematic

The intermediate format is problematic because, while it is syntactically valid JS, it is semantically wrong. By this I mean that the format just happens to be valid JS in both the expression and class property scenarios, for different reasons, but doesn’t actually mean anything. Let’s unpack that.

Take the expression example:

```js
const GreetingExpression = [__GLIMMER_TEMPLATE(`Hello!`, { /*scope*/ })];
```

This intermediate format is a valid JS array that contains a function call (`__GLIMMER_TEMPLATE`). The Babel plugin, provided by ember-template-imports, uses this as a signal, looking for an `ArrayExpression` node type, to transform it to the expression syntax.

Now, let’s take a look at the class property example:

```js
class GreetingClass {
  [__GLIMMER_TEMPLATE(`Hello!`, { /*scope*/ })]
}
```

This is where things get a bit semantically murkier. This is valid JS but actually means nothing. The only reason this works is because it’s valid to define a dynamic class property using square brackets. See this for example:

![image](https://github.com/achambers/ember-template-imports/assets/416724/61e87f25-60fd-4ab6-946d-fee415a23e6b)

So, while the above format is valid JS, it doesn’t make sense to be assigning some dynamic value as a class property with no assigned value. This is semantically incorrect and only works as a happy accident and to serve as a signal for the Babel plugin to match on to transform the code. Ultimately the above intermediate format code, if ran as JS, would output this:

![image](https://github.com/achambers/ember-template-imports/assets/416724/8027a6a9-e81d-419c-b0fa-6aa3013fa898)

Notice, the instance of the class ends up with an `undefined: undefined` property. This makes no sense. (Note, the code in this intermediate format is not actually executed. This is just an example of why the intermediate format is semantically incorrect)

--- 

QUESTION: Why can’t we just go from template tag straight to final Ember output? - Scopes handling…The precompile is just a regex and has no idea about scope of variables accessible to the template…..

TODO: With new approach from RFC, we can use private fields in templates.

---

### What is the new plan of attack?
[RFC#931 - JS Representation of Template Tag](https://github.com/emberjs/rfcs/pull/931) will need to be implemented as library. This will be the replacement for the intermediate format mentioned above and, unlike the intermediate format above, is fully spec compliant and syntactically and semantically correct Javascript that can be run in the browser. This would likely replace the existing APIs of `compileTemplate` and `setComponentTemplate`, or at the very least delegate to them.

---

TODO: Check the accuracy of this statement 👆

---

At this point, people could use these APIs to get first class component template functionality. `<template>` tag just becomes syntactic sugar on top.

As mentioned above, Babel, which is the current lib that transforms our JS code, does not support the ability to plugin into the actual logic that generates the AST. No parser really does. They understand JS and the future versions of JS in order to parse them. The `<template>` tag is custom and not something any of these parsers understand and they maybe never will.

So, for this to be done correctly, we need to fork an existing parser and enhance it to understand the `<template>` tag and be able to convert that into an AST. Ed Faulkner has already done most of this work, forking SWC (Speedy Web Compiler), a Rust based JS/TS compiler, and enhancing it with the ability to parse `<template>` tags.

From there we need to create a plugin for that parser that can take the AST and convert it to the new JS representation mentioned in the RFC above. As this and the compiler are written in Rust, they can be compiled to WASM and shipped as an NPM package which can be included in Ember apps.

At this point Ember apps and addons can import an NPM package for the purpose of parsing gjs/gts files that include `<template>` tags very fast both at build time and run time.

---

A bit airy fairy here 👆 . Not sure how valid these steps are. Need to validate this.

---

### What are the next steps?
In no particular order:

* Complete the implementation of the SWC enhancements and plugin
* Work out how to package, ship and distribute the parser above, as a WASM package
* Tie up the loose ends and final questions around [RFC#931](https://github.com/emberjs/rfcs/pull/931)
* Implement [RFC#931](https://github.com/emberjs/rfcs/pull/931)
* Update ember-template-imports (or create a new addon?) that implements the above as a way to include `<template>` tag support in an Ember app.
* Create great documentation around how to use the template compiler
* Work out how this fits in to Polaris, learning materials and documentation etc etc
  
## References
[RFC: JS Representation of Template Tag](https://github.com/emberjs/rfcs/pull/931)
https://github.com/ef4/content-tag
https://github.com/ef4/swc

