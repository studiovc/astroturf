# css-literal-loader

A webpack loader and babel plugin for extracting and processing css defined in other files.

"Inline css" that just works with CSS, PostCSS, Less, Sass, or any other css preprocessor, and plays nicely with existing style tooling like `extract-text-webpack-plugin`.

```js
import React from 'react';

const styles = css`
  .button {
    color: black;
    border: 1px solid black;
    background-color: white;
  }
`;

export default function Button({ children }) {
  return <button className={styles.button}>{children}</button>;
}
```

When processed, the `css` block will be extracted and treated as a `.css` file, taking advantage of any and all of the other loaders configured to handle css.

It even handles statically analyzable interpolations!

```js
const margin = 10;
const height = 50;
const bottom = height + margin;

const styles = css`
  .box {
    height: ${height}px;
    margin-bottom: ${margin}px;
  }

  .footer {
    position: absolute;
    top: ${bottom}px;
  }
`;
```

### Experimental component API

For those that want something a bit more like styled-components, there is an experimental component API!

```
import { styled, css } from 'css-literal-loader/styled'; // import needed!

const Button = styled('button')`
  color: black;
  border: 1px solid black;
  background-color: white;

  ${props => props.primary && css`
    color: blue;
    border: 1px solid blue;
  `}
`;

render(<Button>A styled button</Button>, mountNode);
```

The above transpiles to

```js
const Button = styled('div', 'Button', require('./FileName-Button.css'), styles => [
  styles.button,
  props => props.primary && styles.buttonVariant1
]
```

Styles are still extracted to a seperate file, and any arrow function intopolations are turned into an array that's passed directly to the react `classNames()` library (with further `css` templates turned into styles references).

There are a whole bucket of caveats of course, to keep the above statically extractable, and limit runtime code.

* We assume you are using css-modules in your css pipeline to return classes from the style files, we don't do any of that ourselves.
* Function interpolations are limited to one level of nesting
* All "top level" styles have any @import statements hoisted up (via a regex)

### WHY?!

The goal of this API is not to mimic or reimplement the features of other css-in-js libraries, but to provide
a more ergonomic way to write normal css/less/sass next to your javascript.

What does that mean? css-in-js libraries are trying to _replace_ css preprocessors to some extent. This means they need to provide ways of doing variables, composition, mixins, imports etc. Usually they do this by leaning
on JS language features where appropraite, and inventing some DSL where needed.

css-literal-loader **doesn't need to do any of that** because it's not trying to replace preprocessors but instead, make it nicer to use **existing** tooling in a Component based context. This means at a minimum we need to help you map styles to your component API (props), which the above does.

#### how do I do all those things then?

Well that really depends on your preprocessor. css-modules as a base provides a great way to compose styles, and
css-literal-loader extracts styles to consistent names;

```js
// Button.js

const helpers = css`
  .heavy {
    font-weight: 900;
  }
`;

const Title = styled('h3')`
  composes: heavy from './Button-helpers.css';

  font-size: 12%;
`;
```

You can also not be afraid to mix and match real `css` files and inline ones. Use Less or Sass mixins and variables, etc.

## Setup

Add the css-literal-loader to JavaScript loader configuration, and whatever you want to handle `.css` files:

```js
{
 module: {
   rules: {
     {
       test: /\.css$/,
       use: [
         'style-loader',
        { loader: 'css-loader', options: { modules: true } }
      ],
     },
     {
       test: /\.js$/,
       use: ['babel-loader','css-literal-loader'],
     },
   }
 }
}
```

### Options

css-literal-loader accepts a few query options.

* **tagName**: (default: `'css'`) The tag identifier used to locate inline css literals and extract them.
* **extension**: (default: `'.css'`) the extension used for extracted "virtual" files. Change to whatever file type you want webpack to process extracted literals as.

**Note:** css-literal-loader expects uncompiled JavaScript code, If you are using babel or Typescript to transform tagged template literals, ensure the loader runs _before_ babel or typescript loaders.

## Use without webpack

If you aren't using webpack and still want to define styles inline, there is a babel plugin for that.

Config shown below with the default options.

```js
// babelrc.js
module.exports = {
  plugins: [
    [
      'css-literal-loader/babel',
      {
        tagName: 'css',
        extension: '.css',
        writeFiles: true, // Writes css files to disk using the result of `getFileName`
        getFileName(hostFilePath, pluginsOptions) {
          const basepath = join(
            dirname(hostFilePath),
            basename(hostFilePath, extname(hostFilePath)),
          );
          return `${basepath}__extracted_style${opts.extension}`;
        },
      },
    ],
  ],
};
```

The extracted styles are also available on the `metadata` object returned from `babel.transform`.

```js
const { metadata } = babel.transformFile(myJsfile);

metadata['css-literal-loader'].styles; // [{ path, value }]
```