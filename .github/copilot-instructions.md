# Internal Developer Guide for Meteor 3.x + Blaze Apps

This documentation provides guidance for working on Meteor-based applications using MeteorJS version 3.x and Blaze as the front-end rendering engine.

---

## Project Basics

* JavaScript dependencies are managed using **npm**.
* The codebase uses **MeteorJS (v3.2.2 or higher)**: [https://www.meteor.com/](https://www.meteor.com/)
* Blaze is used for rendering UI templates: [https://www.blazejs.org/](https://www.blazejs.org/)
* **MongoDB** is the primary database.
* **GitHub** is used for source control and tracking work items.
* **GitHub Actions** handles CI/CD pipelines.

---

## Local Development

Use the latest **LTS version of Node.js (>= 22.x < 24)** when developing locally.

### Commands

Run lint checks:

```bash
meteor npm run lint
```

Start the app:

```bash
meteor npm run start
```

---

## Meteor 3.x Key Differences

Meteor 3.x introduces several enhancements:

### Async/Await Support

All server code should use `async/await`. Meteor methods and database operations now support asynchronous versions.

#### Meteor Method Calls

**Old (2.x):**

```js
Meteor.call('methodName', arg1, arg2, callback)
```

**New (3.x):**

```js
await Meteor.callAsync('methodName', arg1, arg2)
```

#### MongoDB Collection Operations

**Old (2.x):**

```js
Collection.find({}).fetch()
Collection.findOne({})
Collection.insert({})
```

**New (3.x):**

```js
await Collection.find({}).fetchAsync()
await Collection.findOneAsync({})
await Collection.insertAsync({})
```

---

## JavaScript Code Style

Please follow the conventions below to maintain consistency:

* **Strings**: Use single quotes `'like this'`.
* **Indentation**: Use 2 spaces.
* **Variables**: Use `const` and `let`; avoid `var`.
* **String building**: Use template literals `` `Hello ${name}` ``.
* **Functions**: Prefer regular function declarations over arrow functions.
* **Asynchronous Code**: Use `async`/`await`.
* **Modules**: Use `import`/`export` (ES6 modules); avoid CommonJS.
* **Naming**:

  * `camelCase` for variables/functions
  * `PascalCase` for class names
  * `UPPER_CASE` for constants
* Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate.

---

## HTML + Blaze Template Guidelines

* **Attribute helpers** must be enclosed in **single quotes**:

  ```html
  <div class="my-class {{myHelper 'test'}}" title="{{__ 'session_title'}}"></div>
  ```

* **Avoid logic blocks inside HTML tags** (Blaze does not support this):
  ❌ Incorrect:

  ```html
  <div {{#if addClass}}class="my-class"{{/if}}></div>
  ```

  ✅ Instead, handle logic in helpers or use conditionally rendered blocks:

  ```html
  <div class="{{#if addClass}}my-class{{/if}}"></div>
  ```
