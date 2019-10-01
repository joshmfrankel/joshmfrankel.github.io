---
layout: post
title: Composing a reusable compound component
date: 2019-09-25 12:51 -0400
category:
- tutorials
tags:
- React
---

React gives developers a quick and standard way for building highly interactive
components. This makes it easy to pump out features fast and efficiently. However, with great speed, comes great responsiblity 🙄 ... in making sure that you aren't rebuilding the same component over and over again. In essence following common DRY principles.

In this article, we're going to take a look at several iterations of the same component into something
that could be reusable anywhere throughout your application.
<!--excerpt-->

## What we're going to build

For the purposes of this tutorial, we're going to be constructing a drawer component. The
end feature will look like this:

...INSERT GIF...

The first step we'll take is listing out all our high-level feature tasks to complete.

* A clickable title that on click expands an attached content section
* Content section that animates like a drawer
* Screen reader compatible tab functionality

With that we can get started on the component.

## Building the basic React Component

The first thing we'll need is something to store the current state of the container which will either be expanded or collapsed. We can use a state variable `active` to denote this in our component's constructor.

```jsx
import React from 'react';

class Drawer extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      active: false,
    };
  }
}

export default Drawer;
```

We'll also add in the HTML necessary for the component within its render method. `title` will
be a prop directly passed into the component like `<Drawer title="Cool title">` while `children` is a special prop that contains all child elements inside of a component. This means using our component like this:

``` jsx
<Drawer>
  <p>hi</p>
</Drawer>
```

Means that within `Drawer` the `this.props.children` prop returns the inner content of `<p>hi</p>`.

By structuring it in this manner we allow future developers to compose the content they want displayed within the container. It affords a lot of freedom and flexibility in how the component is consumed.

``` jsx
render() {
  return (
    <div className="Drawer">
      <div className="Drawer-title">
        {this.props.title}
      </div>

      <div className="Drawer-content">
        {this.props.children}
      </div>
    </div>
  );
}
```

With a little variable destructing, we can make this even simpler.

``` jsx
render() {
  const { title, children } = this.props;

  return (
    <div className="Drawer">
      <div className="Drawer-title">
        {title}
      </div>

      <div className="Drawer-content">
        {children}
      </div>
    </div>
  );
}
```

Next we need to toggle the above state variable `active` whenever a user clicks the title. We can do this by adding an `onClick` handler to the element.

``` jsx
// This negates the boolean for active which effectively toggles it between true and false
handleTitleClick() {
  this.setState({ active: !this.state.active });
}

render() {
  const { title, children } = this.props;

  return (
    <div className="Drawer">
      <div className="Drawer-title" onClick={this.handleTitleClick}>
        {title}
      </div>

      <div className="Drawer-content">
        {children}
      </div>
    </div>
  );
}
```

Using the active state variable, we can also adjust the CSS classes on the top-level
HTML element. This will let our stylesheet know the current state of the component.

``` jsx
render() {
  const { title, children } = this.props;
  const drawerStyles = this.state.active ? 'is-expanded' : '';

  return (
    <div className={`Drawer ${drawerStyles}`}>
      <div className="Drawer-title" onClick={this.handleTitleClick}>
        {title}
      </div>

      <div className="Drawer-content">
        {children}
      </div>
    </div>
  );
}
```

<blockquote class="Info Info-right">
"setState() does not always immediately update the component. It may batch or defer the update until later. This makes reading this.state right after calling setState() a potential pitfall."
<cite><a href="https://reactjs.org/docs/react-component.html#setstate">- reactjs.org</a></cite>
</blockquote>

There's a minor gotcha at this point with the above setting of state. We're using a negated form of the previous state `!this.state.active` to inform the current state what its value is. The problem is that state updates are batched meaning you can't rely on them being immediate. Because of this we need to explictly use the previous state within the `setState` function.

We'll use an arrow function here to pluck the previous state into a variable named `prevState`. This will allow us to be confident that the state change will be accurate.

``` jsx
handleTitleClick() {
  this.setState(prevState => ({ active: !prevState.active }));
};
```

Now our onClick handler properly deals with existing state.

The onClick handler can only update state by accessing the `this` context. This means we'll also need to bind it to `this` properly within the constructor.

``` jsx
constructor(props) {
  super(props);

  this.state = {
    active: false,
  };

  this.handleTitleClick = this.handleTitleClick.bind(this);
}
```
<blockquote class="Info Info-right">
  "If you are using the experimental public class fields syntax, you can use class fields to correctly bind callbacks"
  <cite>- <a href="https://reactjs.org/docs/handling-events.html">reactjs.org</a></cite>
</blockquote>

This currently works and we can clean this up by taking it one step further using the **public class fields syntax**.

I really love how easy this syntax reads and how it cleans up your constructor function from all the binding cruft.

``` jsx
constructor(props) {
  super(props);

  this.state = {
    active: false,
  };

  // No binding necessary
}

// Instead of () {} is now = () => {}
handleTitleClick = () => {
  this.setState({ active: !this.state.active });
}
```

So with all that out of the way, we now have our first iteration of our component below.

``` jsx
import React from 'react';

class Drawer extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      active: false,
    };
  }

  handleTitleClick = () => {
    this.setState(prevState => ({ active: !prevState.active }));
  };

  render() {
    const { title, children } = this.props;
    const drawerStyles = this.state.active ? 'is-expanded' : '';

    return (
      <div className={`Drawer ${drawerStyles}`}>
        <div
          className="Drawer-title"
          onClick={this.handleTitleClick}
        >
          {title}
        </div>
        <div className="Drawer-content">
          {children}
        </div>
      </div>
    );
  }
}

export default Drawer;

/////////////////////////////////////
// And here's how you would use it //
/////////////////////////////////////
import React from 'react';
import Drawer from '~/components/Drawer'; // The location of our new component

class AnotherComponent extends React.Component {
  // code omitted
  render() {
    return (
      <div className="AnotherComponent">
        <h1>Welcome to my component</h1>
        <Drawer title="My Drawer Title">
          <ul>
            <li>Item 1</li>
            <li>Item 2</li>
            <li>Item 3</li>
          </ul>
        </Drawer>
      </div>
    );
  }
}
```

Here's what it looks like so far. Really the only thing it does is provide
structure and toggle a CSS class `is-expanded` based on state.

![Unstyled Drawer Component](/img/2019/unstyled-drawer-component.gif)

It's pretty ugly at this point, so in the next section we're going to add some
SCSS styles to it.

## Adding Style

One thing missing from our component at the moment is that the Drawer's content
is always shown. We only want it to display if the state of the component is expanded. We're going to eventually animate it into existing using a slide transition so focusing on the element's height will be advantageous here.

Start off by hiding the content using the following style
configuration.

``` scss
.Drawer {
  &-content {
    height: 0;
    overflow: hidden;
  }
}
```

We need both `height: 0` and `overflow: hidden` here as they work in tandem. Having a
height of 0 still shows the content even with a container without height. Adding overflow hidden ensures that the content in the container outside of the height will remain hidden. These two together remove the content.

Next we probably want the `Drawer-title` element to look like it is actually clickable. We can do this by adding a border, background, and cursor styles below.

``` scss
.Drawer {
  &-title {
    background: #efefef;
    border: 1px solid;
    cursor: pointer;
    display: inline-block;
    padding: 16px;
  }

  &-content {
    ...
  }
}
```

The result is a primitive looking button to click. Which brings up something else we should probably fix at this point which is keyboard accessibility. Time for a slight detour.

![Basic title styles](/img/2019/basic-title-style.png)

### Keyboard Accessibility

Right now on the page anyone using a screen reader or other non-mouse assistance technology, will be unable to tab to our new button. One way we could attempt fix this is by giving our `Drawer-title` div a `tabIndex` value and a `role` value.

``` jsx
<div
  className="Drawer-title"
  onClick={this.handleTitleClick}
  tabIndex="0"
  role="button"
>
  {title}
</div>
```

<blockquote class="Info Info-right">
  The eslint plugin jsx-a11y has some great documentation on the subject of non-semantic
  elements having an onClick handler.
  <cite><a href="https://github.com/evcohen/eslint-plugin-jsx-a11y/blob/master/docs/rules/click-events-have-key-events.md" target="_blank">ESlint jsx-a11y</a></cite>
</blockquote>

While, this allows us to focus on the element, the enter and spacebar keys don't activate our `onClick` handler. Major bummer. We could try to add an `onKeyPress` event and capture the spacebar and enter key, but why go the extra effort when there is a perfectly semantic HTML element at our disposal. The `<button>` element.

So let's fix this back in our `render()` method.

``` jsx
render() {
  const { title, children } = this.props;
  const drawerStyles = this.state.active ? 'is-expanded' : '';

  return (
    <div className={`Drawer ${drawerStyles}`}>
      <button
        className="Drawer-title"
        onClick={this.handleTitleClick}
      >
        {title}
      </button>
      <div className="Drawer-content">
        {children}
      </div>
    </div>
  );
}
```

As you can see from the above, we no longer need `tabIndex`, `role`, or even additonal events like `onKeyPress`. We can also remove the `Drawer-title` style for `display: inline-block` since buttons by default are inline-block.

At this point our button just works exactly how we would expect. Hitting the tab key on the page will focus on our new button. Once focussed pressing either enter or spacebar will activate our `onClick` handler.

### Hiding and showing content

Now that we've fixed our button's accesibility issues we can focus back on our goal of showing and hiding content in the Drawer. Since the height of the element starts out as 0 we can simply have its height change to a larger value when the `is-expanded` class is active via the `this.state.active` variable. We want this value to be greater than the containing content's height could ever be. I choose `40vh` here as it is 40% of the current viewports height (and it seemed a reasonable starting place).

``` scss
.Drawer {
  &-title {
    ...
  }

  &-content {
    ...
  }

  &.is-expanded {
    .Drawer-content {
      height: 40vh;
    }
  }
}
```

Making this change now enables the element to show/hide its content. We can make
this even more polished by adding an animated transition to the content.

``` scss
.Drawer {
  &-title {
    ...
  }

  &-content {
    height: 0;
    overflow: hidden;
    transition: height .5s ease;
  }

  &.is-expanded {
    .Drawer-content {
      height: 10vh;
    }
  }
}
```

Now that our `Drawer-content` section animates its height our component is starting
to show off its basic functionality. With a couple more tweaks to the styles below, we've got our first iteration complete.

``` scss
.Drawer {
  width: 300px; // Set parent width
  display: inline-block; // center the element

  &-title {
    padding: 16px;
    border: 1px solid;
    background: #efefef;
    cursor: pointer;
    width: 100%; // Make title button take up full width of .Drawer
  }

  &-content {
    background: #ddd; // Show a visible background
    border-left: 1px solid; // Add borders to all sides except top
    border-right: 1px solid;
    border-bottom: 1px solid;
    height: 0;
    overflow: hidden;
    transition: height .5s ease;
  }

  &.is-expanded {
    .Drawer-content {
      height: 40vh;
    }
  }
}
```

![First iteration on Component](/img/2019/first-iteration-component.gif)

## Static height and variable containers

While our component now functions well enough to show/hide, there is a glaring issue with it. If you look back to our `.is-expanded` class we had to specify a static height of `40vh`. This means that our container (which can contain a dynamic amount of content) will never have a height larger than 40% of the viewport. What if the content is larger than 40%? A lot smaller? Well the results aren't great:

![Static height and variable containers example](/img/2019/set-height-bug.png)

That's a lot of extra space for a single list item!

So how can we fix this? Well, unfortunately if you're looking for a pure CSS solution here the short answer is, you can't. Or at least all the articles I've read online claim there isn't a process and I haven't been able to figure one out either.

Really the only way to determine dynamic height for the content is to calculate it using JavaScript. So let's do just that. Here's what we'll do:

* Remove static height from SCSS
* Calculate the height of the hidden content within the Drawer
* Set the height of the content to the calculated value when visible

### Remove static height from SCSS
We'll start by removing the static height from our SCSS.

``` scss
.Drawer {
  ...

  /* Remove all of this
  &.is-expanded {
    .Drawer-content {
      height: 80vh;
    }
  }*/
}
```

Pretty simple deletion of style code here. Let's keep moving.

### Calculate the height of the hidden content

Next we can calculate the height of the hidden element through some indirection based on the target of the click event. There's a couple ways to accomplish this. Here's the first approach:

``` jsx
handleTitleClick = (event) => {
  const contentHeight = event.target.nextSibling.scrollHeight;
  this.setState(prevState => ({ active: !prevState.active }), () => {
    this.setState({ contentHeight: this.state.active ? contentHeight : 0 });
  });
};
```

There's a bit of added context above so let's break it down line-by-line. First we'll deal with calculating the hidden content's height.

``` jsx
handleTitleClick = (event) => {
  const contentHeight = event.target.nextSibling.scrollHeight;
```

<blockquote class="Info Info-right">"The Element.scrollHeight read-only property is a measurement of the height of an element's content, including content not visible on the screen due to overflow."
<cite>- <a href="https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollHeight" target="_blank">mozilla.org</a></cite>
</blockquote>

We added a new argument `event` that comes from the the click event. Using this we can check its `target` which returns the button's DOM element. `nextSibling` grabs the next related DOM element which in our case is the `Drawer-content` div. `scrollHeight` gives us the element's height and works with elements that are hidden via overflow.

Next we modify our current `setState` call to accept a callback function. Using a callback function ensures that our first call to `this.setState` has finished meaning we can rely directly on any effected state variables again. We do this immediately to toggle our hidden content's height via a ternary clause.

``` jsx
this.setState(prevState => ({ active: !prevState.active }), () => {
  // This second call to setState can rely on this.state.active directly
  // as the previous call must finish before this.
  this.setState({ contentHeight: this.state.active ? contentHeight : 0 });
});
```

Alright, so that's the first attempt. Here's a slightly updated one that is a bit more efficient.

``` jsx
handleTitleClick = (event) => {
  const contentHeight = event.target.nextSibling.scrollHeight;
  this.setState(prevState => ({
    active: !prevState.active,
    contentHeight: !prevState.active ? contentHeight : 0
  }));
};
```

The big change here is that instead of using a callback function with `setState` we're relying on `!prevState.active` to let us know when the current stateful variable `active` is true. It's a bit confusing given that we have to negate it with a `!` but that's because the state starts out as false and then becomes true upon clicking. Additionally, we can't rely on `this.state.active` here because the update to it is happening at the same time (since they are both within the same setState call).

### Setting the height of the content

Now that we've calculated the height of the hidden element, we can use that to directly apply an inline style. This part is actually pretty easy.

{% raw %}
``` jsx
<div
  className="Drawer-content"
  style={{ height: this.state.contentHeight }}
>
  {children}
</div>
```
{% endraw %}

We use an object that contains camelCased CSS attributes as React requires this for dealing with inline styles. For example, if we were working with `max-height` we would instead write the attribute above like: {% raw %}`style={{ maxHeight: this.state.contentHeight }}`{% endraw %}.

With this our component sets the hidden content's height accurately.

![Set Height Component](/img/2019/set-height.gif)

## The dangers of nextSibling

There's still one piece of the above that's bothering me. We're currently finding the element to calculate height based on the following chain of method calls `event.target.nextSibling.scrollHeight`. Well what happens if `nextSibling` doesn't exist? Or if we modify our component to include a new element between the button and the content?

If `nextSibling` doesn't exist then calling `scrollHeight` on undefined is going to raise an exception. Modifying the component means that our content element no longer will be targeted as the next sibling.

``` jsx
return (
  <div className={`Drawer ${drawerStyles}`}>
    <button
      className="Drawer-title"
      onClick={this.handleTitleClick}
    >
      {title}
    </button>

    <div>In-between content section</div>

    <div className="Drawer-content" style={{ height: this.state.contentHeight }}>
      {children}
    </div>
  </div>
);
```

With the new `<div>In-between content section</div>` above we've broken our current height calculation. Now it looks like this:

![Broken Drawer](/img/2019/broken-drawer.gif)

*[DRY]: Don't Repeat Yourself
