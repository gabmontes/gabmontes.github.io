---
title: React in 200 bytes?
---

A few weeks ago, I started thinking on creating a small website to show the relationship between Babel presets and plugins after I realized the documentation in the [Babel's site](babeljs.io/docs/plugins/) is not always up to date.

That website should query a server that just parses the presets' `package.json` files and recursively obtains the related presets. With that information, the web page should render a table. Simple, right?

The idea was to code the server using Node.JS and as few libraries as possible and the website, that should have a single page, using React. At the end, the objective was having some fun and use the code to teach JavaScript, Node.JS and React in the training courses I usually provide to students and organizations.

While working on that, I came across the [Rails Conf 2012 Keynote, "Simplicity Matters", by Rich Hickey](https://www.youtube.com/watch?v=rI8tNMsozo0), and I thought on going one step further. What about trying to redo React in as few lines of code as possible?

I did exactly that some time ago with jQuery, reimplementing some of its basic and most common functionality during a training course on web applications development. The result of that exercise was [`not-jquery`](https://gist.github.com/gabmontes/535a7b3b059b2a301a55b43e90ee0101), a 100-lines file that helped the students understand how to work with the DOM while making jQuery less magical.

## Reimplementing React

In order to start reimplementing React, I needed to set a few restrictions and objectives:

- Support only stateless functional components (no state, lifecycle events, classes)
- Do not use JSX, transpiling, bundling, etc.
- Use no other libraries at all
- Shall work on modern/latest browsers only (no IE)
- Shall be as small as possible (keep only what cannot be taken out)

So, with that restrictions in place, I started playing around with functions, components and the DOM, from the simplest examples to some more complex use cases. In order to give the development path some order, lets follow some of the examples and cases in the [React's Quick Start](https://facebook.github.io/react/docs/hello-world.html) guides.

## Hello World

Everything in programming starts with a "hello world" example, so this was not the exception. The objective was to make this work:

```js
ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
)
```

But I said no JSX! And clearly, what I was developing was not `ReactDOM` so I tweaked it a bit:

```js
notReactDOM.render(
  '<h1>Hello, world!</h1>',
  document.getElementById('root')
)
```

Much better! But how to actually render that piece of HTML in the `root` element? It was simpler than I initially thought: just create a `notReactDOM` global object having a `render` method which do the trick.

```js
notReactDOM = {
  render: function (html, element) {
    element.innerHTML = html
  }
}
```

Simple, very simple!

## Embedding expressions

OK. So far so good. Now it was time to do some magic trying to embed expressions but instead of doing it in the JSX code, in the HTML strings.

```js
function formatName(user) {
  return `${user.firstName} ${user.lastName}`
}

const user = {
  firstName: 'Harper',
  lastName: 'Perez'
}

const greetingComponent = `
  <h1>
    Hello, ${formatName(user)}!
  </h1>
`

notReactDOM.render(
  greetingComponent,
  document.getElementById('root')
)
```

That just worked out of the box. No changes to `notReactDOM.render()` at all! All the magic is in the new [template literals syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals).

## Functional components

But what about inserting some logic, some expressions to change the component content based on the input as when there is no user? Welcome to the functional components:

```js
const greetingComponent = function ({ user }) {
  if (user) {
    return `<h1>Hello, ${formatName(user)}!</h1>`
  }
  return '<h1>Hello, Stranger.</h1>'
}
```

In order to render that component, `notReactDOM.render()` had to be changed to manage both strings or functions returning strings.

```js
notReactDOM = {
  render: function (component, element) {
    element.innerHTML = typeof component === 'function' ? component() : component
  }
}
```

In addition, the call to `render` that was now expecting a function, should properly inject the initial state to the component:

```js
notReactDOM.render(
  () => greetingComponent({ user }),
  document.getElementById('root1')
)
```

It all worked flawlessly!

## The ticking clock

The next challenge was the ticking clock example: render a component and update it once every second.

```js
function tick() {
  function elementComponent() {
    return `
      <div>
        <h1>Hello, world!</h1>
        <h2>It is ${new Date().toLocaleTimeString()}.</h2>
      </div>
    `
  }

  notReactDOM.render(
    elementComponent,
    document.getElementById('root')
  )
}

setInterval(tick, 1000)
```

Did it work? No doubts about it. No changes to our `render` function either.

Of course in this simple reimplementation of React/ReactDOM, the whole DOM will be updated every tick as opposed to the VitualDOM management React does to improve performance (a lot). But it was just an exercise to understand how it all works and what are the most basic components that can make a React application work. Doing VirtualDOM stuff was completely out of scope. And considering there are a few [libraries that implement the diff/patch algorithms](https://github.com/Matt-Esch/virtual-dom), I moved to the next use case.

## Composing components

Given the components return an HTML string, composing components was straightforward: just call a component within other to obtain the corresponding HTML string.

```js
function welcomeComponent({ name }) {
  return `<h1>Hello, ${name}</h1>`
}

function appComponent() {
  return `
    <div>
      ${welcomeComponent({ name: 'Sara' })}
      ${welcomeComponent({ name: 'Cahal' })}
      ${welcomeComponent({ name: 'Edite' })}
    </div>
  `
}

notReactDOM.render(
  appComponent,
  document.getElementById('root')
)
```

Each component is a function and therefore should receive the list of properties needed to customize the rendering. Using the destructuring syntax helps a lot in visualizing the code and making it even more clear.

## Iterating over lists

And what about lists? My original objective was to work over lists of presets and plugins. Remember?

Using `map` to map arrays to their HTML representations is the key to manage iterations.

```js
const numbers = [1, 2, 3, 4, 5]

const listItems = numbers.map(number => `
  <li>${number}</li>
`).join('')

notReactDOM.render(
  `<ul>${listItems}</ul>`,
  document.getElementById('root')
)
```

The important thing when mapping lists to HTML strings is to concatenate those strings before passing it up to the parent component with `''`. Otherwise, JavaScript will convert the arrays to strings by concatenating the elements with `,`.

What about keys to track what items in the list changed? No VirtualDOM, remember? No need for keys.

If a VirtualDOM were implemented, the mapping should be done with a helper function to manage both joining the array elements and keys generation.

## Handling events

The last challenge in this series was to handle events, like the click of a button. In JSX that is pretty easy. Defining a `click` property and binding it to a function in the component.

But in this simple version of React, there was no JSX, and no easy way to bind the event handlers that might be defined inside the components to the resulting HTML that would "live" in the global scope.

```js
let state = {
  isToggleOn: true
}

function setState(newState) {
  Object.assign(state, newState)
  render()
}

function handleClick() {
  setState({
    isToggleOn: !state.isToggleOn
  })
}

function toggleComponent({ isToggleOn }) {
  return `
    <button onclick="handleClick()">
      ${isToggleOn ? 'ON' : 'OFF'}
    </button>
  `
}

function render() {
  notReactDOM.render(
    () => toggleComponent(state),
    document.getElementById('root')
  )
}

render()
```

The solution was hack-ish, clearly not elegant, but it worked again: using global functions that change the global state of the application. The good part of this solution is that going this path forces the developer to use a single global state and use functions that change that state and trigger a new rendering cycle. It had some "redux" smell too. Perhaps the next challenge would be to reimplement is as "not-redux"?

## Conclusion

Reimplementing React was a learning experience for me and for sure will help others to understand there is not magic under the hood but some clever decisions in terms of architecture, data flow and way of thinking the application. At the end, sticking to the initial set of restrictions and taking performance and compatibility aside, the whole code for the library can be summarized as follows:

```js
notReactDOM = {
  render: function (component, element) {
    element.innerHTML = typeof component === 'function' ? component() : component
  }
}
```

And that is less than 200-bytes long!!

## Some resources

- `not-react` experimentation repo: [https://github.com/gabmontes/exploring-not-react](https://github.com/gabmontes/exploring-not-react)
- Babel preset to plugins website: [https://babel-presets.utoctadel.com.ar/](https://babel-presets.utoctadel.com.ar/)
- Server and app code for the website: [https://github.com/gabmontes/babel-preset-to-plugins](https://github.com/gabmontes/babel-preset-to-plugins)
