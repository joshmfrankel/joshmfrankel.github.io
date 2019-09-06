---
layout: post
title: Using callback functions with setState in React
category:
- today-i-learned
---

When working with asynchronous requests and setting state in React, you can
encounter some interesting side-effects. What happens if the request succeeds before
the state is set? The reverse? With situations like this it's hard to be confident
in what the logic flow will look like. setState solves for this via allowing callback
functions to be defined.
<!--excerpt-->

A typical response to create a new User record inside a React component might look something
like this:

``` jsx
createUser = (event) => {
  fetch(`/users`, {
      method: 'post',
      body: JSON.stringify(event)
    })
    .then(response => response.json())
    .then(json => {
      this.setState({ currentUserId: json.id });
    })
}

render() {
  return (
    <div className="MyComponent">
      <form onSubmit={this.createUser}>
        <input type="text" name="userName" />
        <button type="submit" />
      </form>
    </div>
  );
}
```

When the form is submitted, a request to create a new User is sent. Once the
request completes the stateful key currentUserId is updated.

This works well for the current setup. However, what if we want to immediately do
something with the `currentUserId`.

``` jsx
createUser = (event) => {
  fetch(`/users`, {
      method: 'post',
      body: JSON.stringify(event)
    })
    .then(response => response.json())
    .then(json => {
      this.setState({ currentUserId: json.id });
    });

    this.sendWelcomeEmail();
}

sendWelcomeEmail = () => {
  fetch(`/mailers/welcome/${this.state.currentUserId}`)
    .then(response => response.json())
    .then(json => {
      console.log(`Mail sent to: ${this.state.currentUserId}`);
    })
}
```

This won't work because the initial POST request to create the User is
asynchronous and our call to `sendWelcomeEmail` is at the end of the `createUser`
function. That means that `sendWelcomeEmail` will execute possibly before the
`fetch` request returns and `this.state.currentUserId` will be `null`.

Ok, so easy fix right. Just move the `sendWelcomeEmail` into the fetch request.

``` jsx
createUser = (event) => {
  fetch(`/users`, {
      method: 'post',
      body: JSON.stringify(event)
    })
    .then(response => response.json())
    .then(json => {
      this.setState({ currentUserId: json.id });
      this.sendWelcomeEmail(); // moved inside the .then
    });
}

sendWelcomeEmail = () => {
  fetch(`/mailers/welcome/${this.state.currentUserId}`)
    .then(response => response.json())
    .then(json => {
      console.log(`Mail sent to: ${this.state.currentUserId}`);
    })
}
```

Now our `sendWelcomeEmail` is properly waiting until the response from `fetch` is
returned but it still isn't working because `this.state.currentUserId` is null. When
I first encountered this it had me scratching my head. Until I read the following
on [reactjs.org](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous){:target="_blank"}.

> React may batch multiple setState() calls into a single update for performance.<br /><br />
> Because this.props and this.state may be updated asynchronously, you should not rely on their values for calculating the next state.

So, calling `setState` and then directly relying on this.state variables is going to lead
to frustration. Luckily, `setState` allows for callback functions to be supplied which is where our
implementation comes in below.

``` jsx
createUser = (event) => {
  fetch(`/users`, {
      method: 'post',
      body: JSON.stringify(event)
    })
    .then(response => response.json())
    .then(json => {
      this.setState(
        { currentUserId: json.id },
        this.sendWelcomeEmail
      );
    });
}

sendWelcomeEmail = () => {
  fetch(`/mailers/welcome/${this.state.currentUserId}`)
    .then(response => response.json())
    .then(json => {
      console.log(`Mail sent to: ${this.state.currentUserId}`);
    })
}
```

Now our `sendWelcomeEmail` function waits for the fetch request to complete and also
executes as a callback to the state being set for the `currentUserId`. Here's the portion
of code I'm talking about:

``` jsx
this.setState(
  { currentUserId: json.id },
  this.sendWelcomeEmail
);
```

Wrapping up when working with an asynchronous request make sure that your function
is contained with the request (or you use a different Promise configuration). Furthermore,
if your function depends on a stateful variable that could be updated make sure that your
function is listed as a callback to the `setState` call.
