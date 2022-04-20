---
layout: post
title: Simulating a select dropdown change in Jest
categories:
- today-i-learned
tags:
  - jest
  - testing
  - javascript
---

While building a mobile friendly tab component, I created a select dropdown that allowed the user to control
the displayed content. This worked in tandem with the existing click the name of the tab setup. Testing a simple
click event is trivial by using `.click()` but I quickly found that simulating a select dropdown change wasn't as
straightforward nor as documented. I stumbled on the following solution which hopefully helps someone else stuck.
<!--excerpt-->

A typical Jest test for a click event may look like:

```js
it("applies a class to an element", () => {
  const tabs = document.querySelectorAll("[data-tabs-target='tab']")

  // Simulate clicking the third tab
  tabs[2].click()

  expect(tabs[2].className.includes('active')).toEqual(true)
})
```

The above is a good representation of what it looks like for the user to click
the third tab element. 

Ok, so what about a select element?

First off, select elements dispatch a change event. This means that simply clicking
the select and then the option doesn't work. Now there are a couple different methods
for changing the current select elements value.

```js
const select = document.querySelector("#mySelect")

// Change the value
select.value = 'tab 3'

// Change the selected value
select.selected = 'tab 3'

// Change the selected index
select.selectedIndex = 2

// set expectations
```

I found that the `select.value =` option to work best with my scenario. There's a problem here though. Jest doesn't know that the select element has changed. That's because the event wasn't emmitted yet. We can do that by explictly telling the select
element that it triggered a change event.

```js
const select = document.querySelector("#mySelect");

select.value = 'tab 3';
select.dispatchEvent(new Event('change'));

// set expectations
```

With `dispatchEvent(new Event('change'))` the select element now properly sends
the change event with the previously updated value. This simulates a select dropdown
change and allows for testing your JavaScript components which rely upon them.

That's it! Thanks for reading.
