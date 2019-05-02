# Unit Testing React Components

The surface area of a React component has two main sides. On one end, components
declare a PropType contract with which they interact with ancestor components.
They also (typically) render user interfaces, forming a contract with the user
in terms of both what the user sees and what the user can do. Thoroughly unit
testing React components involves covering both of these interfaces. Below, we
will use a simple example component to demonstrate the two main types of unit
tests that can be used to accomplish this goal. The rest of the discussion will
occur in code comments of an annotated unit test suite.

## The CounterBadge

The CounterBadge component displays a [Badge](https://reactstrap.github.io/components/badge/)
and a [Button](https://reactstrap.github.io/components/buttons/). The Badge
starts at 0. Each time the Button is clicked, the Badge increments by one.

![Animation of the CounterBadge Component](./CounterBadge.gif)

_In the animation above, we use the `onIncrement` prop to log to the console._

```javascript
import React from 'react'
import PropTypes from 'prop-types'
import { Badge, Button } from 'reactstrap'

class CounterBadge extends React.Component {
  state = {
    count: 0,
  }

  increment = () => {
    const count = this.state.count + 1
    this.setState({ count })
    this.props.onIncrement(count)
  }

  render() {
    return (
      <React.Fragment>
        <Badge color={this.props.color}>{this.state.count}</Badge>
        <Button color={this.props.color} onClick={this.increment}>
          Add One
        </Button>
      </React.Fragment>
    )
  }
}

CounterBadge.propTypes = {
  color: PropTypes.oneOf(['primary', 'secondary']).isRequired,
  onIncrement: PropTypes.func.isRequired,
}

export default CounterBadge

```

### Testing The CounterBadge

```javascript
import React from 'react'
import { shallow } from 'enzyme'
import CounterBadge from './CounterBadge'
import { Badge, Button } from 'reactstrap'

/*
  Unit tests for React components fall into one of two categories:

    1. "Render Tests" or "View Tests" which set up a certain prop and state
      combination (given) and then expect some result to be rendered (then).
      These tests may look for the existence of children, the correct props on
      those children, or other things related to how the component renders.
    2. "Behavioral Test" or "Interaction Tests", which set up a certain prop
      and state combination (given), perform some user action (when), and then
      assert the resulting effects **in terms of callbacks called or state
      updated.** These tests do NOT directly test what is rendered after the
      action.
  CounterBadge has both types of tests.
*/
describe('CounterBadge', () => {
  const render = ({ color = 'secondary', onIncrement = () => {} } = {}) =>
    shallow(<CounterBadge color={color} onIncrement={onIncrement} />)

  /*
    Test suites are divided up by shared setup, not by shared behavior. Notice,
    for example, that we have a `describe` block for the default state of 0, as
    well as a `describe` for the a higher starting state. Especially as
    components get large, it can be tempting to add `describe` blocks for new
    functionality. This is typically a code smell indicating that a smaller
    component can be factored out.
  */
  describe('default', () => {
    let wrapper
    let onIncrement

    beforeEach(() => {
      onIncrement = jest.fn()
      wrapper = render({ onIncrement })
    })

    /*
      This is an example of the first type of test, a render test. Given the
      initial setup (in this case props and default state), we assert that a
      given child, the Badge, is rendered. We also assert that the Badge is
      displaying the right thing by checking its `children` prop.

      Note that we are using `shallow` rendering instead of `mount` in this
      suite. `shallow` not only exhibits much better performance, but it also
      helps maintain a separation of responsibilities between parent and child
      components. Child components should be treated as though they were native
      DOM elements:  atomic, reusable components with well-defined and well-
      tested behavior.

      In this case, we trust that Badge is rendering its children. We do not
      care how, as that is a concern internal to the Badge.
    */
    it('starts counting at zero', () => {
      expect(wrapper.find(Badge).props().children).toBe(0)
    })

    /*
      This is another example of a render test.
    */
    it('has the correct text on the Button', () => {
      expect(wrapper.find(Button).props().children).toBe('Add One')
    })

    /*
      These next two tests are examples of the second type of test, which
      perform some action and assert the resulting state changes or side
      effects.

      Note first that we are performing this action by finding the appropriate
      method on the child and calling it directly. This is similar to using the
      Enzyme 'simulate' method on a native DOM component. By calling the prop
      method directly, our test ensures that the method not only performs the
      correct action; it also ensures that we are passing it to the right child.

      A common alternative is to call an internal method on the component under
      test. In this case, `wrapper.instance().increment()`. This should be
      avoided. Using this instance-based approach is brittle to refactors such
      as renaming of the instance method or composing methods in different ways
      that achieve the same functionality.

      **
      **Direct use of `.instance()` should typically be considered a red flag!**
      **
    */
    it('increments the count by one when the Button is clicked', () => {
      const onClick = wrapper.find(Button).props().onClick
      onClick()
      expect(wrapper.state().count).toBe(1)
    })

    /*
      This is another example of a behavioral/interaction test. In this case,
      after the action is performed, we assert that a callback prop has been
      called with the correct arguments. By testing this callback prop here,
      we do not have to separately assert that the `onClick` method is passed to
      the Button, or that the Button `typeof onClick` is 'function'... it covers
      one side of the contract (user interaction) all the way to the other (prop
      type contract).
    */
    it('calls back with the updated count', () => {
      const onClick = wrapper.find(Button).props().onClick
      onClick()
      expect(onIncrement).toHaveBeenCalledWith(1)
    })
  })

  /*
    In our second describe suite, we set up the component with different state.
  */
  describe('with a higher count', () => {
    let wrapper
    let onIncrement

    beforeEach(() => {
      onIncrement = jest.fn()
      wrapper = render({ onIncrement })
      wrapper.setState({ count: 17 })
    })

    it('displays the current count on the Badge', () => {
      expect(wrapper.find(Badge).props().children).toBe(17)
    })

    /*
      Note that in these interaction tests that we are asserting the resulting
      state of the component. Specifically, we are NOT asserting that it renders
      the correct props on the next render cycle. Since we check that the state
      is set directly, and elsewhere we check that given the direct state we
      render the correct results, we are covering the full cycle. Each unit test
      can stay small and focused on the piece that counts.

      **Don't mix the two types of tests into one! Keep them simple!**
    */
    it('increments the count by one when the Button is clicked', () => {
      const onClick = wrapper.find(Button).props().onClick
      onClick()
      expect(wrapper.state().count).toBe(18)
    })

    it('calls back with the updated count', () => {
      const onClick = wrapper.find(Button).props().onClick
      onClick()
      expect(onIncrement).toHaveBeenCalledWith(18)
    })
  })

  /*
    These tests around the color props are examples of further render/view tests.
    In this case, the shared setup is the `color` prop. Note that these tests
    do not cover the increment behavior, which is assumed to work the same
    regardless of color. Assumptions like this can be helpful in keeping the
    test code concise, but be careful when the component becomes large. When
    there are many props that customize the component's behavior, it can be
    difficult to test all cases thoroughly. In that case, break it out into
    smaller components with fewer props each.
  */
  describe('with primary color', () => {
    let wrapper

    beforeEach(() => {
      wrapper = render({ color: 'primary' })
    })

    it('renders a Badge with primary color', () => {
      expect(wrapper.find(Badge).props().color).toBe('primary')
    })

    it('renders a Button with primary color', () => {
      expect(wrapper.find(Button).props().color).toBe('primary')
    })
  })

  describe('with secondary color', () => {
    let wrapper

    beforeEach(() => {
      wrapper = render({ color: 'secondary' })
    })

    it('renders a Badge with secondary color', () => {
      expect(wrapper.find(Badge).props().color).toBe('secondary')
    })

    it('renders a Button with secondary color', () => {
      expect(wrapper.find(Button).props().color).toBe('secondary')
    })
  })
})
```
