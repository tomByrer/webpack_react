# Webpack and React

Facebook's [React](https://facebook.github.io/react/) is one of those projects that has changed the way we think about frontend development. Thanks to [React Native](https://facebook.github.io/react-native/) the approach isn't limited just to web. Although simple to learn, React provides plenty of power.

Webpack is an ideal tool to complement it. By now we understand how to set up a simple project on top of Webpack. Let's turn it into a React project next and implement a little todo app. It won't be very complex but will help you to understand some of the basics.

## Installing React

To get started install React to your project. Just hit `npm i react --save` and you should be set. As a next step we could port our **app/component.js** to React. Provided we use ES6 module and class syntax and JSX, we can go with a solution like this:

**app/component.js**

```javascript
import React from 'react';

export default class Hello extends React.Component {
    render() {
        return <h1>Hello world</h1>;
    }
}
```

In addition we'll need to adjust our `main.js` to render the component correctly. Here's one solution:

**app/main.js**

```javascript
import React from 'react';
import Hello from './component';

React.render(<Hello />, document.getElementById('app'));
```

> I strongly advise against rendering directly to `document.body`. This can cause strange problems with plugins that rely on body due to the way React works. It is much better idea to give it a container of its own.

## Setting Up Webpack

In order to make everything work again, we'll need to tweak our configuration a little. In order to deal with ES6 and JSX, we'll use [babel-loader](https://www.npmjs.com/package/babel-loader). Install it using `npm i babel-loader --save-dev`. In addition add the following loader declaration to the *loaders* section of your configuration:

```javascript
{
    test: /\.js$/,
    loader: 'babel',
    include: path.join(ROOT_PATH, 'app'),
}
```

We will specifically include our `app` source to our loader. This way Webpack doesn't have to traverse whole source. Particularly going through `node_modules` can take a while. You can try taking `include` statement out to see how that affects the performance.

> An alternative would be to `exclude` but `include` feels like a better solution here.

If you hit `npm run build` now, you should get some output after a while. Here's a sample:

```bash
> webpack_demo@1.0.0 build /Users/something/projects/webpack_demo
> webpack --config config/build

Hash: 00f0b93cf085fc55d4ec
Version: webpack 1.7.3
Time: 1149ms
    Asset    Size  Chunks             Chunk Names
bundle.js  633 kB       0  [emitted]  main
   [0] multi main 28 bytes {0} [built]
    + 162 hidden modules
```

As you can see, the output is quite chunky in this case! Don't worry. This is just an unoptimized build. We can do a lot about the size at a later stage when we apply optimizations, minification and split things up.

## Resolving JSX Files

The configuration above works but what if we want to resolve files with `.jsx` extension? After all it can be a good idea to separate vanilla JavaScript files from those containing JSX.

To achieve this we need to extend out Regex pattern like this:

```javascript
{
    test: /\.jsx?$/,
    loader: 'babel',
    include: path.join(ROOT_PATH, 'app'),
}
```

If you try renaming **app/component.js** as **app/component.jsx** and hit `npm run build`, you should get an error like this:

```bash
ERROR in ./app/main.js
Module not found: Error: Cannot resolve 'file' or 'directory' ./component in /Users/something/projects/webpack_demo/app
 @ ./app/main.js 11:13-35
```

This means Webpack cannot resolve our `import Hello from './component';` to a file.

The problem has to do with Webpack's default resolution settings. Those settings describe where Webpack looks for modules and how. We'll need to tweak these settings a little.

Add the following bits to your configuration:

```javascript
var common = {
    ...
    resolve: {
        extensions: ['', '.js', '.jsx', '.css'],
    }
};

exports.build = {
    ...
    resolve: common.resolve,
};

exports.develop = {
    ...
    resolve: common.resolve,
};
```

Now Webpack will be able to resolve files ending with `.jsx` and everything should be fine. If you try running `npm run build` again, the build should succeed.

## Activating Hot Loading for Development

If you hit `npm run dev` and try to modify our component (make it output `hello world again` or something), you'll see it actually works. After a flash. We can get something fancier with Webpack, namely hot loading. This is enabled by [react-hot-loader](https://gaearon.github.io/react-hot-loader/).

To make this work, you should `npm i react-hot-loader --save-dev` and tweak the configuration as follows. The configuration has been included in its entirety so you don't have to piece it together:

**config/index.js**

```javascript
var path = require('path');
var webpack = require('webpack');

var ROOT_PATH = path.resolve(__dirname, '..');

var common = {
    entry: [path.join(ROOT_PATH, 'app/main.js')],
    output: {
        path: path.resolve(ROOT_PATH, 'build'),
        filename: 'bundle.js',
    },
    module: {
        loaders: [
            {
                test: /\.css$/,
                loaders: ['style', 'css'],
            },
        ],
    },
};

exports.build = {
    entry: common.entry,
    output: common.output,
    module: {
        loaders: common.module.loaders.concat([
            {
                test: /\.jsx?$/,
                loader: 'babel',
                include: path.join(ROOT_PATH, 'app'),
            }
        ])
    }
};

exports.develop = {
    entry: common.entry.concat(['webpack/hot/dev-server']),
    output: common.output,
    module: {
        loaders: common.module.loaders.concat([
            {
                test: /\.jsx?$/,
                // inject react-hot loader for development here
                loaders: ['react-hot', 'babel'],
                include: path.join(ROOT_PATH, 'app'),
            }
        ])
    },
    plugins: [
        // hot module replacement plugin itself. if you pass `--hot` to
        // webpack-dev-server, do not activate this!
        new webpack.HotModuleReplacementPlugin()
        // do not reload if there is a syntax error in your code
        new webpack.NoErrorsPlugin()
    ]
};
```

Note what happens if you `npm run dev` now and try to modify the component. There should be no flash (no refresh) while the component should get updated provided there was no syntax error.

The advantage of this approach is that the user interface retains its state. This can be quite convenient! There will be times when you may need to force a refresh but this tooling decreases the need for that significantly.

XXX: discuss eslint + jsx extension now?

## Implement Basic Todo

Now that we have some nice tooling, we can actually get productive with it...

XXXXX

## Optimizing Rebundling

XXX: this is difficult to set up with react-hot-loader. see https://github.com/gaearon/react-hot-loader/blob/master/docs/README.md#usage-with-external-react . maybe skip this part?

As waiting for the reload to happen can be annoying we can take our approach a little further. Instead of making Webpack go through React and all its dependencies, you can override the behavior in development. We'll set up an alias for it and point to its minified version.

**webpack.config.js**

```javascript
var path = require('path');
var node_modules = path.resolve(__dirname, 'node_modules');
var pathToReact = path.resolve(node_modules, 'react/dist/react.min.js');

var config = {
  entry: path.resolve(__dirname, 'app/main.js'),
  resolve: {
    alias: {
      'react': pathToReact
    }
  },
  output: {
    path: path.resolve(__dirname, 'build'),
    filename: 'bundle.js'
  },
  module: {
    noParse: [pathToReact]
  }
};

module.exports = config;
```
We do two things in this configuration:

1. Whenever "react" is required in the code it will fetch the minified React.js file instead of going to *node_modules*

2. Whenever Webpack tries to parse the minified file, we stop it, as it is not necessary

Take a look at `Optimizing development` for more information on this.

> TBD: point at the correct chapter here

## Type Checking with Flow

If you come to JavaScript from other programming languages you are familiar with types. You have types in JavaScript too, but you do not have to specify these types when declaring variables, receiving arguments etc. This is one of the things that makes JavaScript great, but at the same time not so great.

Specifically when working on very large projects with many developers type checking gives stability to your project, much like a good test does. So using **Flow** is definitely not a requirement. It is for developers who depends on type checking as more of a routine and for the before mentioned large projects with many developers. Webpack makes it easy to include **Flow** in your workflow.

### Installing flow

- Have to try this out :-)
- What about "flowcheck-loader", tried it? https://www.npmjs.com/package/flowcheck-loader (probably works, haven't tried this one yet)
- https://tryflow.org/

> TBD: expand this section

## PureRenderMixin

This gives you a very short and nice syntax for defining components. A drawback with using classes though is the lack of mixins. That said, you are not totally lost. Lets us see how we could still use the important **PureRenderMixin**.

```javascript
import React from 'react/addons';

class Component extends React.Component {
  shouldComponentUpdate() {
    return React.addons.PureRenderMixin.shouldComponentUpdate.apply(this, arguments);
  }
}

class MyComponent extends Component {
  constructor() {
    this.state = {message: 'Hello world'};
  }
  render() {
    return (
      <h1>{this.state.message}</h1>
    );
  }
}
```