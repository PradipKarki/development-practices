# Composition over Customization

In large React applications, complexity can often become difficult to manage.
How should components and component hierarchies be extended to support new
functionality? The OOP principal of [Composition over Inheritance](
https://en.wikipedia.org/wiki/Composition_over_inheritance) still applies to
React components. Additionally, Composition should be preferred over
Customization as well.

## What is Customization?

React makes it easy to create user interfaces using declarative components. Some
components are pretty straightforward:

```javascript
const Title = ({ text }) => <h1>{text}</h1>

<Title text="My Title" />
```

As components get more complex, props may be added for different parameters:

```javascript
const Title = ({ text, Heading = h1, type }) => {
  const formattedText = type === 'italic' ? <em>{text}</em> : text
  return <Heading>{formattedText}</Heading>
}
```

These additional props provide customization options for the component.
Depending on the props provided, the component may behave in subtly different
ways. In some cases, props trigger entirely new modes of operation! With
increased customization, sufficiently covering each combination of props in unit
tests also becomes more challenging.

### Is that Title example really so bad?

No, not on its own. It is a simple example to show what customization looks
like. Even using this simple component multiple times across an application
could be unwieldy. Consider using composition to make it nicer for common cases:

```javascript
const ItalicTitle = ({ text }) => <Title type="italic" text={text} />
const SmallTitle = ({ text }) => <Title heading={h3} text={text} />

<ItalicTitle text="This title is italic" />
<SmallTitle text="Smaller title" />
```

## An Example Application

![Fruit App Usage Animation](./Fruit%20App.gif)

Let's step into an example application to see how this principal can be applied
in a more complex scenario. In this application, we have a page for each of our
favorite fruits, and links to navigate between them.

```javascript
import React from 'react'
import PropTypes from 'prop-types'
import { Link } from 'react-router-dom'

const Page = ({ fruit }) => {
  let title
  if (fruit === 'apples') {
    title = "How do you like them apples?"
  } else {
    title = `Welcome to the ${fruit} page!`
  }

  let main
  switch (fruit) {
    case 'apples':
      main = 'An apple a day keeps the doctor away.'
      break
    case 'oranges':
      main = 'A healthy dose of Vitamin C to prevent scurvy!'
      break
    case 'grapes':
      main = 'Fruit of the vine, I like it just fine.'
      break
    default:
  }

  return (
    <div>
      <h1>{title}</h1>
      <h2>Fruit Pages</h2>
      <Sidebar />
      <h2>About the Fruit</h2>
      <p>{main}</p>
    </div>
  )
}

Page.propTypes = {
  fruit: PropTypes.string.isRequired,
}

export default Page
```

To tie this together, we use a simple React Router `Switch`:

```javascript
import React from 'react';
import { Redirect, Router, Route, Switch } from 'react-router-dom';
import { createBrowserHistory } from 'history'
import { ApplesPage, OrangesPage, GrapesPage } from './FruitPages'
import './App.css';

function App() {
  return (
    <div className="App">
      <Router history={createBrowserHistory()}>
        <Switch>
          <Route exact path="/apples" render={() => <Page fruit="apples" />} />
          <Route exact path="/oranges" render={() => <Page fruit="oranges" />} />
          <Route exact path="/grapes" render={() => <Page fruit="grapes" />} />
          <Route><Redirect to="/apples" /></Route>
        </Switch>
      </Router>
    </div>
  );
}
export default App;
```

This is a perfect example of a component hierarchy using Customization. The
`Page` component is used for each `Route`, and a prop is passed in which
controls the overall behavior of the `Page`.

### Testing With Customization

In order to unit test `Page`, we would want to test the basic layout (`h1`,
`h2s`, `Sidebar` all exist). We would also add `describe` blocks for each
possible `fruit` option and expect the title and main sections to be rendered
correctly for those fruits. And... do we add those layout tests for each fruit?
We may want to, since a bug with one of the fruit could lead to an
incorrect layout.

That's a lot of test cases for a simple component!

### Extending Customized Components

One day, the Product Owner requests that you add their new favorite fruit to the
page: mangoes!

To do this, you have to add:
1. A Route for `/mangoes`
2. A sidebar option for `/mangoes`
3. Title logic for mangoes. Does it use the default, or a custom title?
4. Main text logic for mangoes
5. `Page` tests for the above logic (and probably copy the tests for the layout!)

At this point, `Page` is starting to look somewhat large. Most of the logic has
silent defaults that won't warn us if we forget to add, for example, the right
title for our new fruit. Most importantly, why are we touching a `Page`
component when nothing about the `Page` layout itself is changing?

## Refactoring to use Composition

Similar to our `<Title />` refactor, let's restruture this application to make
better use of composition. We will start with a Page that only handles the
page layout.

```javascript
import React from 'react'
import PropTypes from 'prop-types'
import Sidebar from './Sidebar'

const Page = ({ title, main }) => {
  return (
    <div>
      {title}
      <Sidebar />
      <h2>About the Fruit</h2>
      <p>{main}</p>
    </div>
  )
}

Page.propTypes = {
  title: PropTypes.node.isRequired,
  main: PropTypes.string.isRequired,
}

export default Page
```

Let's compose this in different ways to get our fruit pages:

```javascript
import React from 'react'
import PropTypes from 'prop-types'
import Title from './Title'
import Page from './Page'

const DefaultTitle = ({ fruit }) => <Title title={`Welcome to the ${fruit} page!`} />
DefaultTitle.propTypes = {
  fruit: PropTypes.string.isRequired,
}

// For brevity, this example includes all of these in the same file.

export const ApplesPage = () => (
  <Page
    title={<Title title="How do you like them apples?" />}
    main="An apple a day keeps the doctor away."
  />
)
export const GrapesPage = () => (
  <Page
    title={<DefaultTitle fruit="grapes" />}
    main="Fruit of the vine, I like it just fine."
  />
)
export const OrangesPage = () => (
  <Page
    title={<DefaultTitle fruit="oranges" />}
    main="A healthy dose of Vitamin C to prevent scurvy!"
  />
)
```

And finally, our updated App Router:

```javascript
import React from 'react';
import { Redirect, Router, Route, Switch } from 'react-router-dom';
import { createBrowserHistory } from 'history'
import { ApplesPage, OrangesPage, GrapesPage } from './FruitPages'
import './App.css';

function App() {
  return (
    <div className="App">
      <Router history={createBrowserHistory()}>
        <Switch>
          <Route exact path="/apples" render={() => <ApplesPage/>} />
          <Route exact path="/oranges" render={() => <OrangesPage />} />
          <Route exact path="/grapes" render={() => <GrapesPage />} />
          <Route><Redirect to="/apples" /></Route>
        </Switch>
      </Router>
    </div>
  );
}

export default App;
```

### Testing with Composition

How would we test these components? The App component is about the same as
before, needing to test each route for its rendered page.

The FruitPages are new. Each page have its own test suite verifying that the
correct title and expected main text are set. Note that we do not include any
tests about page layout here.

The Page component would need tests only for the layout. Specifically, it has
no knowledge of fruit any more! This is a huge win! We have managed to separate
out our fruit logic from our layout logic and avoided duplicating tests for
each one.

### Extending Composed Components

Now, to add our Product Owner's favorite fruit, our path is clear:

1. Create a `MangoesPage` component with the correct title and main text
2. Add a Route for `/mangoes` which renders a MangoesPage
3. Add a Sidebar link for `/mangoes`
4. Add unit tests for MangoesPage, the new Route, and the new Sidebar link

We didn't have to touch Page at all, and we could add in Mangoes without risk of
interfering with our other fruit pages. All of our Mango related functionality
was encapsulated into a `MangoesPage` component, which _composed_ with `Page` to
acheive shared layout functionality. Success!
