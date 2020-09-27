# Vue.js, ESLint and Prettier - [Why can't we be friends?](https://youtu.be/nmhgi665Oek?t=165)

## Motivation etc.

(?)

## Life goals
- have a consistent (and pleasant!) `eslint` linting experience in VSCode, `vue-cli-service` and on the command line, while working with a Vue.js project
- use `prettier` to auto-format code on save or from the command line
- make `eslint` and `prettier` behave like friends, not enemies fighting over your code

I know this all sounds very ambitious (especially the last one!), but I promise we'll get there!

We're gonna follow a hands-on approach, first playing around to get a feel of the behavior, and then digging into why and how things work behind the covers.

## Getting our feet wet

A quick list of the tools and versions I'm using for this tutorial(?):
- Visual Studio Code 1.49.2
- VSCode extensions:
  - Vetur 0.28.0
  - ESLint 2.1.8  
- @vue/cli 4.5.6 (used to create a bare-bone project)

I'm also using Git Bash from [Git for Windows](https://git-scm.com/download/win) as a terminal, because, well, I'm old and grumpy and still using Windows.

We'll start with a very simple Vue app created by running `vue create vue-simple-app` with the following settings:

```
Vue CLI v4.5.6
? Please pick a preset: Manually select features
? Check the features needed for your project: Choose Vue version, Babel, Linter
? Choose a version of Vue.js that you want to start the project with 2.x
? Pick a linter / formatter config: Basic
? Pick additional lint features: Lint on save
? Where do you prefer placing config for Babel, ESLint, etc.? In dedicated config files
? Save this as a preset for future projects? No
```

## First achievement: VSCode linting expert

If you're already using VSCode, you can use this trick(?) to start it with a clean slate while keeping your current settings untouched.

For this tutorial(?) I'm using the following minimal VSCode configuration:
```json
{
    "telemetry.enableCrashReporter": false,
    "telemetry.enableTelemetry": false,
    "workbench.list.openMode": "doubleClick",
    "terminal.integrated.shell.windows": "C:\\Program Files\\Git\\bin\\bash.exe",
    "editor.tabSize": 2,
    "files.eol": "\n",
    "terminal.integrated.rendererType": "experimentalWebgl"
}
```

I also (only) have `Vetur` and `ESLint` extensions installed.

Let's open the app folder in VSCode and open the `src/Vue.app` file. We should see a dialog asking us if we want to use our project's local `eslint` version from `node_modules`:

![](/images/eslint-1.png)

This allows us to use different `eslint` versions for different projects, so of course we're gonna accept!

Let's now see what happens if we make the most common (and probably the most hated, too) code inaccuracy (because it's not really a mistake) - declaring an unused variable: add `const a = 10` on a new line right under the `import`. We can immediately see a red squiggly under `a`, signaling that there's something wrong with our code. Hovering over it we get:

![](/images/eslint-2.png)

From `eslint(no-unused-vars)` we can deduce it's an `eslint` error because we're breaking the `no-unused-vars` rule. If a rule sounds weird to you (they usually do!) and want to know more you can click on "Quick Fix" -> "Show documentation for ...". Doing that in our case gets us to the [`no-unused-vars` rule documentation page](https://eslint.org/docs/rules/no-unused-vars).

Let's now check that linting also works inside the `<template>` block: add `<div v-for="n in 5">{{ n }}</div>` on a new line right under `<HelloWorld>`, we should see another red squiggly and on hover:

![](/images/eslint-3.png)

(I'll leave it to you to figure out what this error is about).

You've probably already noticed that we get the error twice and thought "What's up with that?!". One thing that it's not immediately obvious is that [`vetur` also validates](https://github.com/vuejs/vetur/blob/master/docs/linting-error.md) `<template>`, `<script>` and `<style>` blocks in SFCs, and we end up getting the same error twice - one from `eslint` itself, and the other from `vetur`, using `eslint-plugin-vue` to validate the `<template>` block. This happens for historical reasons, back when `eslint` didn't know how to directly parse and validate `.vue` files, and it was `vetur`'s responsibility to parse them and run `eslint` separately for the found `<template>` and `<script>` blocks. Because `eslint` now supports parsing `.vue` files and validating `<template>` and `<script>` blocks, we should disable this redundant checking by `vetur`:

```jsonc
// VSCode settings.json
{
  // ...
  "vetur.validation.script": false,
  "vetur.validation.template": false
}
```

(I usually place this project-dependent settings in the VSCode's Workspace config, that's stored in the `.vscode/settings.json` file, in the project folder. I also version them.)

We'll still keep `vetur`'s `<style>` validation for now, which uses [VSCode's built-in CSS validation](https://code.visualstudio.com/docs/languages/css#_syntax-verification-linting). You can disable it using `"vetur.validation.style": false`.

If we did everything right, we should now see the previous error only once:

![](/images/eslint-4.png)

And so we get our first achievement unlocked: VSCode linting expert! Actually there's tree more ways to lint your code that should now work out-of-the-box. We'll take a look at them next.

### Linting from the terminal

You can manually lint all your project files by running `yarn lint` (or `npm run lint`, for the nostalgic) inside your project folder. Make sure you saved the two changes we did to `App.vue` and try it:

```sh
$ yarn lint
yarn run v1.22.5
$ vue-cli-service lint
error: Elements in iteration expect to have 'v-bind:key' directives (vue/require-v-for-key) at src\App.vue:5:5:
  3 |     <img alt="Vue logo" src="./assets/logo.png">
  4 |     <HelloWorld msg="Welcome to Your Vue.js App"/>
> 5 |     <div v-for="n in 5">{{ n }}</div>
    |     ^
  6 |   </div>
  7 | </template>
  8 |


error: 'a' is assigned a value but never used (no-unused-vars) at src\App.vue:11:7:
   9 | <script>
  10 | import HelloWorld from './components/HelloWorld.vue'
> 11 | const a = 10
     |       ^
  12 |
  13 | export default {
  14 |   name: 'App',


2 errors found.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is very useful to be run as a commit hook so that teams can stay confident that all committed code is consistent with the agreed-upon linting rules. It can also be used to auto-fix the whole project code-base at once. We'll talk more about auto-fixing in a bit.

### Linting during development

Let's now try to run the dev-server. First undo the two `App.vue` changes we did earlier, save and run`yarn serve`. The dev-server should start and the app should be accessible at `http://localhost:8080/`. Open it in a browser and let's reintroduce the `const a = 10` inaccuracy. You'll notice we get an error from the dev-server:

```sh
 ERROR  Failed to compile with 1 errors                                            18:17:04
 error  in ./src/App.vue

Module Error (from ./node_modules/eslint-loader/index.js):

D:\work\vue-simple-app\src\App.vue
  10:7  error  'a' is assigned a value but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)


 @ ./src/main.js 6:0-28 10:13-16
 @ multi (webpack)-dev-server/client?http://192.168.1.16:8080&sockPath=/sockjs-node (webpack)/hot/dev-server.js ./src/main.js
```

and a similar error overlay in the browser:

![](/images/eslint-5.png)

If we now remove the error and save everything should go back to normal - the app compiles and the error overlay goes away.

This usually annoy the hell out of people (and eventually make them hate `eslint` and never use it again in their life, before giving up on web development entirely) when e.g. commenting out a piece of code and getting lots of `no-unused-vars` errors and that dreaded error overlay, so they now have to find and comment all those unused declarations before things go back to normal and they can continue coding away. We'll see how we can elegantly tackle this a bit later, but until then, if you really want to feel better and disable `eslint` in the dev-server, create a `vue.config.js` in the project folder (if not already there) and set [`lintOnSave`](https://cli.vuejs.org/config/#lintonsave) like this:

```js
// vue.config.js
module.exports = {
  // ignore linting errors while in development
  // fail production build if there are linting errors
  lintOnSave: process.env.NODE_ENV === 'production' ? 'error' : false
}
```

This will only disable linting while in development, as I think it's still valuable to have it enabled when building a production bundle.

### Linting when building a production bundle

By default (and with the `lintOnSave` configuration from the previous section), running a build with `yarn build` fails if there are any linting errors:

```sh
$ yarn build
yarn run v1.22.5
$ vue-cli-service build
$ D:\work\vue-simple-app\node_modules\.bin\vue-cli-service build

/  Building for production...

 ERROR  Failed to compile with 1 errors                                                     18:49:57
 error  in ./src/App.vue

Module Error (from ./node_modules/eslint-loader/index.js):

D:\work\vue-simple-app\src\App.vue
  10:7  error  'a' is assigned a value but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)


 @ ./src/main.js 6:0-28 10:13-16
 @ multi ./src/main.js

 ERROR  Build failed with errors.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is useful in CI, to do the linting and building in the same step vs. having an additional linting step.

### ESLint configuration

Now that we saw how `eslint` is supposed to behave with a default Vue CLI configuration, let's see how we can bend it to our will. While `eslint` supports a [handful of configuration file types](https://eslint.org/docs/user-guide/configuring#configuration-file-formats), Vue CLI uses `.eslintrc.js`, which by default has the following contents:

```js
// .eslintrc.js
module.exports = {
  root: true,
  env: {
    node: true
  },
  'extends': [
    'plugin:vue/essential',
    'eslint:recommended'
  ],
  parserOptions: {
    parser: 'babel-eslint'
  },
  rules: {
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'warn' : 'off'
  }
}
```

Why a `.js` config? Because it allows us to programmatically generate the config. This can be very useful if we want to enable, disable or change rules config depending on the environment, like we can see above.

The most important part (and the only one we'll be fiddling with) of the `eslint` config is the linting rules. Instead of listing all the rules in our project's config, we can extend preset configs carefully crafted by people smarter than us. We can see the default Vue CLI `eslint` config extends two such presets: [`node_modules/eslint-plugin-vue/lib/config/essential.js`](https://unpkg.com/browse/eslint-plugin-vue@6.2.2/lib/configs/essential.js) and [`node_modules/eslint/conf/eslint-recommended.js`](https://unpkg.com/browse/eslint@6.8.0/conf/eslint-recommended.js). If we want to customize these presets, we can override the rules config with the `rules: { ... }` object. We can see in the default config the rules `no-console` and `no-debugger` are turned off in development, and configured to generate a warning when building for production (note that, by default, the production build will still fail if we use any `console.log()` or `debugger` statements in our code, ever if they're configured as warnings).

We can get the list of all available `eslint` rules [here](https://eslint.org/docs/rules/) and of Vue.js (added by the `eslint-plugin-vue` package) [here](https://eslint.vuejs.org/rules/).

Because trying to figure out which `eslint` rules are active at any point is quite tedious (not to say error-prone), there's an `eslint` helper command that allows us to see the exact config `eslint` uses for linting a certain file: `yarn eslint --print-config filename`. Let's check which Vue.js rules are currently enabled while linting our `App.vue` file:

```sh
# Vue.js rules begin with 'vue/'
# grep -A2 shows two more line after the matched one
$ yarn eslint --print-config src/App.vue | grep 'vue/' -A2
    "vue/no-async-in-computed-properties": [
      "error"
    ],
    "vue/no-dupe-keys": [
      "error"
    ],
[...]
```

### Some better defaults, maybe?

(?)

```js
// .eslintrc.js
function disableInDevelopment(rules) {
  return rules.reduce((acc, rule) => {
    // warn in development, error in production
    // to completely disable in development, use 'off' instead of 'warn'
    acc[rule] = process.env.NODE_ENV === 'production' ? 'error' : 'warn';
    return acc;
  }, {});
}

module.exports = {
  root: true,
  env: {
    node: true,
  },
  extends: ['plugin:vue/recommended', 'eslint:recommended', '@vue/prettier'],
  parserOptions: {
    parser: 'babel-eslint',
  },
  rules: {
    ...disableInDevelopment(['no-console', 'no-debugger', 'no-unused-vars']),
    // other custom rules here
  },
};
```

- use of `plugin:vue/recommended`

### Fix me, baby, one more time!

As I said earlier, some linting rules can be auto-fixed. To see how this works, let's clear all errors from `App.vue` and add a new line right under `import` - `const re = new RegExp(/  /)`. You'll notice that besides `no-unused-vars` we get a new error, `no-regex-spaces`. If we hover over it and click on "Quick Fix..." we see a new menu option "Fix this no-regex-spaces problem":

![](/images/eslint-6.png)

If we click on it the error goes away and our newly added code is replaced with `const re = new RegExp(/ {2}/)`. Magic! There aren't that many rules that support auto-fixing, but this will come in handy when meeting `prettier` in a bit.

To go a step further, because clicking on those small links and menus to fix things is annoying, we can configure VSCode to do that for us every time we save:

```jsonc
// VSCode settings.json
{
  // ...
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

Now if we try to re-add the initial code and save, VSCode should fix all auto-fixable rules for us.

### Introducing Prettier - your handsome code pal

(?)

- `yarn add --dev prettier eslint-plugin-prettier @vue/eslint-config-prettier`

```js
// .eslintrc.js
module.exports = {
  // ...
  'extends': [
    'plugin:vue/essential',
    'eslint:recommended',
    '@vue/prettier'
  ],
  // ...
}
```
