# Ember.js Style Guide

## Table Of Contents

* [General](#general)
* [Organizing your modules](#organizing-your-modules)
* [Models](#models)
* [Controllers](#controllers)
* [Templates](#templates)
* [Routing](#routing)
* [Ember data](#ember-data)


## General

### Import what you use, do not use globals

Future versions of Ember will be released as ES2015 modules, so we'll be
able to import `Ember.computed` directly as `computed`. Use the
[`ember-cli-shims`](https://github.com/ember-cli/ember-cli-shims) module until
Ember has been released as a full ES2015 modules project.

```javascript
// Good

import Model from 'ember-data/model';
import attr from 'ember-data/attr';

import computed, { alias } from 'ember-computed';

const { computed } = Ember;
const { alias } = computed;

export default Model.extend({
  firstName: attr('string'),
  lastName: attr('string'),

  surname: alias('lastName'),

  fullName: computed('firstName', 'lastName', function() {
    // Code
  })
});

// Bad

export default DS.Model.extend({
  firstName: DS.attr('string'),
  lastName: DS.attr('string'),

  fullName: Ember.computed('firstName', 'lastName', {
    get() {
      // Code
    },

    set() {
      // Code
    }
  }),

  fullNameBad: function() {
    // Code
  }.property('firstName', 'lastName')
});
```

### Use `get` and `set`

Calling `someObj.get('prop')` couples your code to the fact that
`someObj` is an Ember Object. It prevents you from passing in a
POJO, which is sometimes preferable in testing. It also yields a more
informative error when called with `null` or `undefined`.

Although when defining a method in a controller, component, etc. you
can be fairly sure `this` is an Ember Object, for consistency with the
above, we still use `get`/`set`.

```js
// Good
import get from 'ember-metal/get';
import set from 'ember-metal/set';

set(this, 'isSelected', true);
get(this, 'isSelected');

// Bad

this.set('isSelected', true);
this.get('isSelected');
```

## Organizing your modules

```js
export default Component.extend({
  // Defaults
  tagName: 'span',

  // Single line CP
  post: alias('myPost'),

  // Multiline CP
  authorName: computed('author.firstName', 'author.lastName', {
    get() {
      // Code
    },

    set() {
      // Code
    }
  }),

  // Lifecycle hooks
  didReceiveAttrs() {
    this._super(...arguments);
    // code
  },

  actions: {
    someAction() {
      // Code
    }
  }
});
```

- Define your object's default values first.
- Define single line computed properties (`thing: alias('myThing')`) second.
- Multi-line computed properties should follow your single line CPs. Please
  follow the [new computed syntax](http://emberjs.com/blog/2015/05/13/ember-1-12-released.html#toc_new-computed-syntax).
- Define lifecycle hooks (`init`, `didReceiveAttrs`, `didRender`, `willDestroy`)
  after computed properties.
- Define your route/component/controller's action last, to provide a common
  place that actions can be found.

### Override init

Rather than using the object's `init` hook via `on`, override init and
call `_super` with `...arguments`. This allows you to control execution
order. [Don't Don't Override Init](https://dockyard.com/blog/2015/10/19/2015-dont-dont-override-init)

### Use Pods structure

Store local components within their pod, global components in the
`components` structure.

```
app
  application/
    template.hbs
    route.js
  blog/
    index/
      blog-listing/ - component only used on the index template
        template.hbs
      route.js
      template.hbs
    route.js
    comment-details/ - used within blog templates
      component.js
      template.hbs

  components/
    tag-listing/ - used throughout the app
      template.hbs

  post/
    adapter.js
    model.js
    serializer.js
```

## Models

### Organization

Models should be grouped as follows:

* Attributes
* Associations
* Computed Properties

Within each section, the attributes should be ordered alphabetically.

```js
// Good

import Ember from 'ember';
import Model from 'ember-data/model';
import attr from 'ember-data/attr';
import { hasMany } from 'ember-data/relationships';

const { computed } = Ember;

export default Model.extend({
  // Attributes
  firstName: attr('string'),
  lastName: attr('string'),

  // Associations
  children: hasMany('child'),

  // Computed Properties
  fullName: computed('firstName', 'lastName', function() {
    // Code
  })
});

// Bad

import Ember from 'ember';
import Model from 'ember-data/model';
import attr from 'ember-data/attr';
import { hasMany } from 'ember-data/relationships';

const { computed } = Ember;

export default Model.extend({
  children: hasMany('child'),
  firstName: attr('string'),
  lastName: attr('string'),

  fullName: computed('firstName', 'lastName', function() {
    // Code
  })
});

```

## Controllers

### Define query params first

For consistency and ease of discover, list your query params first in
your controller. These should be listed above default values.

### Do not use ObjectController or ArrayController

ObjectController is presently deprecated, and ArrayController will be as
well. Controllers are not going to be used in Ember 2.0, so by using
`Controller`, you will make it easier to migrate to 2.0.

### Alias your model

It provides a cleaner code to name your model `user` if it is a user. It
is more maintainable, and will fall in line with future routable
components

```javascript
export default Controller.extend({
  user: alias('model')
});
```

## Templates

### Do not use partials

Always use components. Partials share scope with the parent view, use
components will provide a consistent scope.

### Use block syntax

Use block syntax instead of `in` syntax with block helpers

```hbs
{{! Good }}
{{#each posts as |post|}}

{{! Bad }}
{{#each post in posts}}
```

### Use components in `{{#each}}` blocks

Contents of your each blocks should be a single line, use components
when more than one line is needed. This will allow you to test the
contents in isolation via unit tests, as your loop will likely contain
more complex logic in this case.

```hbs
{{! Good }}
{{#each posts as |post|}}
  {{post-summary post=post}}
{{/each}}

{{! Bad }}
{{#each posts as |post|}}
  <article>
    <img src={{post.image}} />
    <h1>{{post.title}}</h2>
    <p>{{post.summar}}</p>
  </article>
{{/each}}
```

## Routing

### Route naming
Dynamic segments should be underscored. This will allow Ember to resolve
promises without extra serialization
work.

```js
// bad
this.route('foo', { path: ':fooId' });

// good
this.route('foo', { path: ':foo_id' });
```

[Example with broken
links](https://ember-twiddle.com/0fea52795863b88214cb?numColumns=3).

### Perform all async actions required for the page to load in route `model` hooks

The model hooks are async hooks, and will wait for any promises returned
to resolve. An example of this would be models needed to fill a drop
down in a form, you don't want to render this page without the options
in the dropdown. A counter example would be comments on a page. The
comments should be fetched along side the model, but should not block
your page from loading if the required model is there.

## Ember Data

### Be explicit with Ember Data attribute types

Even though Ember Data can be used without using explicit types in
`attr`, always supply an attribute type to ensure the right data
transform is used.

```javascript
// Good

export default Model.extend({
  firstName: attr('string'),
  jerseyNumber: attr('number')
});


// Bad

export default Model.extend({
  firstName: attr(),
  jerseyNumber: attr()
});
```

### Organize your models

Group attributes, relations, then computed properties. Organize each
subgroup alphabetically.
