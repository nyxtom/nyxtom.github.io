---
title: Drawing and editing markdown text with canvas
published: true
description: We will use the canvas api and some very basic parsing to write a simple text editor
tags: [javascript,tutorial,webdev]
series: Writing a canvas markdown editor
---

![COVER](/assets/f9ft70y9u3qffbj09nf1.png)

This last week I've been mucking about with the [canvas api](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Basic_usage). I've put together some visualizations and went through my old content on [p5.js](/2019/08/18/noise/) (where I go into length on flow fields and noise algorithms: check it out, I really enjoyed that one).

In my playing around I've been putting together some ideas around graphing tools and decided one of the most basic things users need in a graph tool is the ability to type in a text input. There are a number of ways to do this, including overlaying HTML on top of a canvas drawing surface (or using [d3.js](https://d3js.org/)). Instead, I chose to just write a simple script that uses the existing canvas api. Like all things, there is more to it than meets the eye, but if you're just trying to get things started - well, here we go.

## Setting up our project

To start, you'll need an HTML and a bit of CSS to setup our sample code. It's not much, but obviously it's a starting point.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Map</title>
    <link rel="stylesheet" href="index.css">
    <script type="text/javascript" src="load.js"></script>
</head>
<body>
    <canvas></canvas>
</body>
</html>
```

In a separate file for css I've setup a few basic reset variables and some root styling. It's not really totally necessary, but I like having these things when I start out.


```css
/** index.css */
:root {
    --root-font-size: 12px;
    --bg: #fafafa;
    --text-color: #333333;
}

/** Reset */
html, body, nav, ul, h1, h2, h3, h4, a, canvas {
    margin: 0px;
    padding: 0px;
    color: var(--text-color);
}
html, body {
    font-family: Roboto, -apple-system, BlinkMacSystemFont, 'Segoe UI', Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    font-size: var(--root-font-size);
    background: var(--bg);
    height: 100%;
    width: 100%;
    overflow: hidden;
}
*, body, button, input, select, textarea, canvas {
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    outline: 0;
}
```

One of the things I really like about the latest CSS is that you don't really need any build tools for it. You can get the most out of your webapp with just [root variables](https://developer.mozilla.org/en-US/docs/Web/CSS/:root). Often, on small projects like these I don't go much further than that - just some root variables and I'm good.

There's actually a great post on how to do complete turing logic in CSS using these variables. Check it out, the author actually made a full minesweeper game using the ["Space Toggle" technique](https://twitter.com/James0x57/status/1282303255826046977).

## Canvas API

> The **canvas** element creates a fixed-size drawing surface that exposes one or more rendering contexts, which are used to create and manipulate the content shown. In this tutorial, we focus on the 2D rendering context. Other contexts may provide different types of rendering; for example, WebGL uses a 3D context based on OpenGL ES.

Create a file `load.js` with the following

```javascript
/** load.js */
var canvas, context;
var text = [''];

function setup() {
    canvas = document.querySelector('canvas');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    context = canvas.getContext('2d');
    context.font = '18px Roboto';
}

function draw() {
    /* draw code */
}

window.onresize = function () {
    if (canvas) {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }
}

window.onkeypress = function (e) {
}

window.onkeydown = function (e) {
}

window.onload = function () {
    setup();
}
```

Couple things going on here. First, we're waiting until the window loads via [onload](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) meaning we have waited until all resources have been loaded.

Once **setup** has been called, we grab the canvas and set it to the window height/width. We ensure that the width/height is also set when the window resizes via the [onresize event](https://developer.mozilla.org/en-US/docs/Web/API/GlobalEventHandlers/onresize).

## Key press / Key Down

Since this is an editor, we want to presumably write something when the keys are pressed. Update the **onkeypress** and **onkeydown** code to the following:

```javascript
window.onkeypress = function (e) {
    if (e.key === 'Enter') {
        text.push('');
    } else {
        text[text.length - 1] += e.key;
    }
    draw();
}

window.onkeydown = function (e) {
    if (e.key === 'Backspace' && text.length && text[0].length) {
        let txt = text[text.length - 1];
        txt = txt.slice(0, txt.length - 1);
        text[text.length - 1] = txt;
        if (!txt.length && text.length > 1) {
            text = text.slice(0, text.length - 1);
        }
    }
    draw();
}
```

These functions are effectively going to manage our text state. **It isn't comprehensive**, but for the moment we can do basic things like typing and hitting enter / backspace to make changes to our text array.

## Drawing

Let's get to the draw code. Whenever we are in canvas, it is proper to clear the screen first before you make additional draw changes. In visualizations and generative art, you can take advantage of what is already there to create some neat effects. But since we're drawing text on every key stroke and update, then we want to clear the screen and refresh the content as such.

```javascript
function draw() {
    context.clearRect(0, 0, window.innerWidth, window.innerHeight);

    let offset = 0;
    let totalHeight = 0;
    let height = (18 * 1.5); // font * line height

    let items = text.map(txt => {
        let width = context.measureText(txt).width;
        let item = {
            txt,
            width,
            offset
        };
        offset = offset + height;
        totalHeight += height;
        return item;
    });

    let cY = (window.innerHeight / 2) - (totalHeight / 2);
    items.forEach(item => {
        let x = window.innerWidth / 2 - item.width / 2;
        let y = item.offset + cY;
        context.fillText(item.txt, x, y);
    });
}
```

![Editor](/assets/dyuik5kja6mej2wbljn6.gif)

In the above code here, we are using the canvas api's [**measureText**](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/measureText). There are alternative methods to measuring text here if we want to be even more precise such as offloading the text into another dom element using the [getBoundingBoxClientRect](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect). I've chosen the canvas method for now as we will end up taking advantage of the rendering context below to make additional measurements.

In any case, we have ourselves a minimal text input with support for multiple lines and backspacing. Let's carry on!

## Markdown

Since this is supposed to be a markdown editor. Markdown as a spec is fairly minimal, but we aren't going to get to all of it in one post. I'll leave you to expand on this, but for now we will implement just the headings portion of the spec.

To do this, we'll need a few things to parse out our text lines and then swap out our calls to context as appropriate.

Add the following code to parse the text line

```javascript
function parse(txt) {
    let lineHeight = 1.5;
    let headingSize = 32;
    let baseSize = 16;
    if (txt.trim().startsWith('#')) {
        let level = txt.match(/\s*\#/g).length;
        let size = headingSize - (level * 4);
        return {
            font: `bold ${size}px roboto`,
            height: size * lineHeight,
            txt
        };
    } else {
        return {
            font: `${baseSize}px roboto`,
            height: baseSize * lineHeight,
            txt
        };
    }
}
```

Then in the draw code update it to call our **parse** function.

```javascript
function draw() {
    context.clearRect(0, 0, window.innerWidth, window.innerHeight);

    let offset = 0;
    let totalHeight = 0;

    let items = text.map(txt => {
        let item = parse(txt);
        item.offset = offset;
        offset = offset + item.height;
        totalHeight += item.height;
        return item;
    });

    let centerY = (window.innerHeight / 2) - (totalHeight / 2);
    items.forEach(item => {
        context.font = item.font;
        let width = context.measureText(item.txt).width;
        let x = window.innerWidth / 2 - width / 2;
        let y = item.offset + centerY;
        context.fillText(item.txt, x, y);
    });
}
```

Notice, that we have moved the **measureText** code into the code right before we actually attempt to draw it. This is because we have changed the rendering context on the line prior to it with the `context.font = item.font`. We want to be sure we make the right measurements based on the current rendering context.

![Markdown](/assets/46y3u1iphpopsunhfbkq.gif)

## Conclusion

There you have it! It's quite basic and minimal, but it's as good a start as any.
I'll leave it to you to fill in more of the code to finish the rest of the spec.
