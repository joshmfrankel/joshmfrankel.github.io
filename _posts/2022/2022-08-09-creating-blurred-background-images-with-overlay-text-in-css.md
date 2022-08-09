---
layout: post
title: Creating blurred background images with overlay text in CSS
date: 2022-08-09 08:18 -0400
categories:
- articles
tags:
  - css
---

Blurring images is a common style on the web. It can give needed contrast and visual
interest. Now you can pre-blur an image with a photo editing program, but for the
purpose of this article we'll be focused on CSS approaches.

<!--excerpt-->

## A Simple Animated Blur

The first type of blur is for background images that **don't** have content overlays. They are simply an image that needs to be blurred out.
This can be useful for blurring an image and then bringing it into focus when the hover state is activated. Try hovering over the following background image.

<style>
  .demo-container {
    outline: 1px solid #000;
    width: 150px;
  }
  .simple-background {
    height: 150px;
    width: 150px;
    background-image: url(/img/2022/brave-browser.png);
    background-size: cover;
    filter: blur(30px);
    transition: 100ms filter linear;
  }
  .simple-background:hover {
    filter: none;
  }
</style>

<div class="demo-container">
  <div class="simple-background"></div>
</div>

This is achieved by using the following css:

```css
.simple-background {
  height: 150px;
  width: 150px;
  background-image: url(/img/2022/brave-browser.png);
  background-size: cover;
  filter: blur(30px);
  transition: 100ms filter linear;
}
.simple-background:hover {
  filter: none;
}
```

First we set a Gaussian blur on the element using `filter: blur(30px);`. Next after we've set the url for the background-image we supply`background-size: cover;`. This scales the image to fit the container and helps o avoid a tiling effect. Next a `transition: 100ms filter linear;` on the starting style to allow for the element to transition its blur effect. Last, `filter: none;` is used when hovering over the element.

## A blurred image with overlay text

Things become more complex if you want to blur the background while also having a text overlay. Let's take a look at the following example:

<style>
  .content-example {
    background-image: url(/img/2022/brave-browser.png);
    height: 150px;
    width: 150px;
    background-size: cover;
    position: relative;
  }
  .content-example--badBlur {
    filter: blur(30px);
  }
  .content-example p {
    color: #fff;
    background: #ccc;
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    margin: 40% 0%;
  }
</style>

<div class="demo-container">
  <div class="content-example">
    <p>Sample content</p>
  </div>
</div>

```html
<div class="content-example">
  <p>Sample content</p>
</div>
```

We have content that needs to sit on top of a background-image. Watch what happens
if we try to blur the background-image by adding `filter: blur(30px);` to the .content-example class.

<div class="demo-container">
  <div class="content-example content-example--badBlur">
    <p>Sample content</p>
  </div>
</div>

The inner text also became blurred. This might be obvious since we blurred the parent
element that all children would also become blurred. So, how do we blur the background-image
without blurring the text?

That's where **backdrop-filter** comes in handy.

### Introducing backdrop filter

Backdrop-filter allows you to blur the background behind an element instead of the element
itself. Now in order for this to work with background-image we'll need to structure our
html so that the parent div has the background-image while the inner has the backdrop-filter. Then the inner div needs to fill the remaining width and height to overlay the image.

```html
<div class="advanced__background">
  <div class="advanced__backgroundBody">
    <p>Sample content</p>
  </div>
</div>
```

We require one additional div to add the backdrop-filter attribute. From the above
example `advanced__backgroundBody` blurs the background by utilizing the width and height of
its parent element. So while `advanced__backgroundBody` doesn't have any color applied to it
the element sits in the same position as `advanced__background` where the background-image is
located. This allows for blurring to occur in the background while preserving the foreground text "Sample content". Below is the matching css for the new HTML structure:

```css
.advanced__background {
  background-image: url(/img/2022/brave-browser.png);
  height: 150px;
  width: 150px;
  background-size: cover;
}
.advanced__backgroundBody {
  position:  relative;
  height: 100%;
  width: 100%;
  backdrop-filter: blur(30px);
  transition: 100ms backdrop-filter linear;
}
.advanced__backgroundBody:hover {
  backdrop-filter: none;
}
.advanced__backgroundBody p {
  color: #fff;
  background: #ccc;
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  margin: 40% 0%;
}
```

Similiar to the simple blur, we need nearly the same css. The main difference as mentioned above
is that the child div (**advanced__backgroundBody**) now has `backdrop-filter: blur(30px); width: 100%; height: 100%;` set on it. The filter and sizing work together to blur out anything behind the element. We even have a transition for the backdrop while hovering. Here's the final result:

<style>
  .advanced__background {
    background-image: url(/img/2022/brave-browser.png);
    height: 150px;
    width: 150px;
    background-size: cover;
  }
  .advanced__backgroundBody {
    position:  relative;
    height: 100%;
    width: 100%;
    backdrop-filter: blur(30px);
    transition: 100ms backdrop-filter linear;
  }
  .advanced__backgroundBody:hover {
    backdrop-filter: none;
  }
  .advanced__backgroundBody p {
    color: #fff;
    background: #ccc;
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    margin: 40% 0%;
  }
</style>

<div class="demo-container">
  <div class="advanced__background">
    <div class="advanced__backgroundBody">
      <p>Sample content</p>
    </div>
  </div>
</div>

And that's it! You now have a background image which is blurred out for effect
while retaining the legibility of the foreground text.

As always, thanks for reading.
