---
title: Simulating webkit force, canvas color swatches
published: true
description: Where we update our realtime drawing app in canvas to support a javascript color swatch and a simple keyboard based force emulation
tags: [javascript,tutorial,webdev]
series: Realtime collaborative drawing with canvas and WebRTC
---

![COVER](/assets/akfa09vndrna72umogaa.gif)

I promised to give our [drawing tool in canvas](/2020/09/08/collaborative-drawing-sse-webrtc/) an upgrade, so let's get a look at that now. There's a few things that it currently doesn't allow such as:

* Switching colors
* Pressure values only available in webkit
* Single line based brush tool

## Color Swatch

From our [previous article](/2020/09/08/collaborative-drawing-sse-webrtc/) our color generation was simply a random function on load. I looked at a number of color generations and libraries and they all were quite a bit too much. Specifically many of these color picker libraries add code for converting between different color schemes like CMYK, RGB, HSL, HSV. Instead, an easier (one function) approach will be to simply create a swatch (similar to Google Docs).

All we need really to accomplish this is have an array of defined colors.

```javascript
const swatch = [
    ["#000000", "#434343", "#666666", "#999999", "#b7b7b7", "#cccccc", "#d9d9d9", "#efefef", "#f3f3f3", "#ffffff"],
    ["#980000", "#ff0000", "#ff9900", "#ffff00", "#00ff00", "#00ffff", "#4a86e8", "#0000ff", "#9900ff", "#ff00ff"],
    ["#e6b8af", "#f4cccc", "#fce5cd", "#fff2cc", "#d9ead3", "#d0e0e3", "#c9daf8", "#cfe2f3", "#d9d2e9", "#ead1dc"],
    ["#dd7e6b", "#ea9999", "#f9cb9c", "#ffe599", "#b6d7a8", "#a2c4c9", "#a4c2f4", "#9fc5e8", "#b4a7d6", "#d5a6bd"],
    ["#cc4125", "#e06666", "#f6b26b", "#ffd966", "#93c47d", "#76a5af", "#6d9eeb", "#6fa8dc", "#8e7cc3", "#c27ba0"],
    ["#a61c00", "#cc0000", "#e69138", "#f1c232", "#6aa84f", "#45818e", "#3c78d8", "#3d85c6", "#674ea7", "#a64d79"],
    ["#85200c", "#990000", "#b45f06", "#bf9000", "#38761d", "#134f5c", "#1155cc", "#0b5394", "#351c75", "#741b47"],
    ["#5b0f00", "#660000", "#783f04", "#7f6000", "#274e13", "#0c343d", "#1c4587", "#073763", "#20124d", "#4c1130"]
];
```

The above color swatch should relatively correspond to the same colors you might see in Google Docs. Next, we need to generate a list of divs that will correspond to the given colors in this swatch.

```javascript
const colorMap = swatch.flat();

let swatchContainer = document.querySelector('#color-picker');
let colorElements = {};
swatch.forEach(row => {
    let rowElem = document.createElement('div');
    rowElem.classList.add('hstack');
    row.forEach(c => {
        let elem = document.createElement('div');
        elem.classList.add('box');
        elem.style.backgroundColor = c;
        colorElements[c] = elem;
        rowElem.appendChild(elem);
    });

    swatchContainer.appendChild(rowElem);
});
```

Simple! Now let's go ahead and add a click behavior to this code such that the box will become active/inactive according to the current color.

```javascript
elem.onclick = function (e) {
    colorPicker.dataset.color = c;
    colorPicker.style.color = c;
    if (colorElements[color]) {
        colorElements[color].classList.remove('active');
    }
    color = c;
    elem.classList.toggle('active');
    e.preventDefault();
};
```

Recall that our random color was generating a color based on the **rgb** and used `Math.random()` to do this. We will replace that code with the following in order to generate a random color within the existing swatch.

```javascript
function randomColor() {
    return parseInt(Math.random() * colorMap.length);
}

var colorIndex = randomColor();
var color = colorMap[colorIndex];
var colorPicker = document.querySelector('[data-color]');
colorPicker.dataset.color = color;
colorPicker.style.color = color;
colorElements[color].classList.add('active');
```

Great! Let's get ahead and add the html to correspond to this setup. Ideally, our color picker should behave as a dropdown. Let's add the initial html there.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Let's Draw Together</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/remixicon@2.5.0/fonts/remixicon.css">
    <link rel="stylesheet" href="/static/index.css">
    <link rel="alternate icon" type="image/png" href="/static/logo.png">
    <link rel="icon" type="image/svg+xml" href="/static/logo.png">
</head>
<body>
    <div class="flush vstack">
        <div class="menubar hstack">
            <a class="icon-link center">
                <i class="ri-lg ri-landscape-line"></i>
            </a>
            <div class="spacer"></div>
            <a class="icon-link active center" data-tool="pencil">
                <i class="ri-lg ri-pencil-fill"></i>
            </a>
            <a class="icon-link center" data-tool="rect">
                <i class="ri-lg ri-shape-line"></i>
            </a>
            <a class="icon-link center" data-tool="circle">
                <i class="ri-lg ri-checkbox-blank-circle-line"></i>
            </a>
            <a class="icon-link center" data-tool="text">
                <i class="ri-lg ri-font-size-2"></i>
            </a>
            <div class="spacer"></div>
            <div class="relative">
                <a class="icon-link center" data-color="#33ffff">
                    <i class="ri-lg ri-palette-line"></i>
                    <i class="ri-lg ri-checkbox-blank-fill center"></i>
                </a>
                <div id="color-picker" class="dropdown vstack">
                </div>
            </div>
            <div class="spacer"></div>
        </div>
        <div class="spacer app">
            <canvas></canvas>
        </div>
    </div>

    <script type="text/javascript" src="/static/load.js"></script>
    <script type="text/javascript" src="/static/draw.js"></script>
</body>
</html>
```

Notice that we are wrapping the **data-color** link in a `relative` class. Let's make sure we have the corresponding classes to handle this.

```css
.relative {
    position: relative;
}
```

The behavior that I would like to replicate is that the dropdown shows up on hover (either hover in the swatch or hover on this data-color link).

```css
:root {
    /** .... */
    --dropdown-background: #fff;
    --dropdown-shadow:  0px 0px 1px 0px rgba(0,0,0,0.5), 0px 2px 6px -5px rgba(0,0,0,0.75);
}

.dropdown {
    position: absolute;
    background-color: var(--dropdown-background);
    padding: 4px;
    box-shadow: var(--dropdown-shadow);
    border-radius: 4px;
    z-index: 1;
    margin-left: -80px;
    transition: all 0.25s ease-in-out;
}
.icon-link + .dropdown {
    opacity: 0;
    top: 8px;
    visibility: hidden;
    transition: all 0.25s ease-in-out;
}
.icon-link:hover + .dropdown, .dropdown:hover {
    opacity: 1;
    top: 16px;
    visibility: visible;
}
.icon-link:hover + .dropdown *, .dropdown:hover * {
    opacity: 1;
}
```

Finally, we need our **box** classes to handle hover/layout.

```css
/** Color Picker */
.box {
    width: 24px;
    height: 24px;
    cursor: pointer;
}
.box:hover, .box.active {
    box-shadow: inset 0px 0px 2px 2px #fff;
    opacity: 0.75;
}
```

Great! Now we have ourselves a color swatch tool!

![Color Swatch](/assets/lhd7ozqozmcjbbdwzsh6.gif)

## Simulating Webkit Force

Recall that if you are in Safari you can use the `webkitForce` property on the [onwebkitmouseforcechanged](https://developer.mozilla.org/en-US/docs/Web/API/Element/webkitmouseforcechanged_event) event to get the current trackpad pressure value. Since this is proprietary we have no real way of accessing this value without either working in Safari (or in Swift for desktop apps). We can however, simulate this kind of value by using a keypress value to increase or decrease a force value. This will somewhat mimic a change as we are moving around our cursor.

```javascript
var force = 1;
var mouseDown = false;

function move(e) {
    mouseDown = e.buttons;
    /** ... */
}

function key(e) {
    if (e.key === 'Backspace') {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
    }
    if (mouseDown && e.key === 'ArrowUp') {
        force += 0.025;
    }
    if (mouseDown && e.key === 'ArrowDown') {
        force -= 0.025;
    }
}

window.onkeydown = key;
```

Now whenever we press the key **up** or **down** we can change the value while we are pressing down on the mouse!

![Force Brush](/assets/qj4nuen7swihdrnc0q3z.gif)

## Changing Colors with Left/Right Arrow Keys

One additional thing I'd like to add is the ability to change the colors on keypress as well. We can simply update the current color index to do this.

```javascript
function key(e) {
    if (e.key === 'Backspace') {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
    }
    if (e.key === 'ArrowRight') {
        colorIndex++;
    }
    if (e.key === 'ArrowLeft') {
        colorIndex--;
    }
    if (e.key === 'ArrowRight' || e.key === 'ArrowLeft') {
        if (colorIndex >= colorMap.length) {
            colorIndex = 0;
        }
        if (colorIndex < 0) {
            colorIndex = colorMap.length - 1;
        }
        if (colorElements[color]) {
            colorElements[color].classList.remove('active');
        }
        color = colorMap[colorIndex];
        colorPicker.dataset.color = color;
        colorPicker.style.color = color;
        colorElements[color].classList.toggle('active');
    }
    if (mouseDown && e.key === 'ArrowUp') {
        force += 0.025;
    }
    if (mouseDown && e.key === 'ArrowDown') {
        force -= 0.025;
    }
}
```

![Switching Colors](/assets/e6kz0a444zw6r0anngcm.gif)

What about combining them at the same time? To do this, we'll need to change our keys for force changes to something like **SHIFT** and **ALT**. We still want to be able to control with the up/down arrows and we want to limit shift/alt to only when left or right is pressed.

```
    if (mouseDown && (e.key === 'ArrowUp' || (e.shiftKey && ['ArrowLeft', 'ArrowRight'].includes(e.key)))) {
        force += 0.025;
    }
    if (mouseDown && (e.key === 'ArrowDown' || (e.altKey && ['ArrowLeft', 'ArrowRight'].includes(e.key)))) {
        force -= 0.025;
    }
```

![Pressure and Color](/assets/d0byzjtlep5qfnoqurk2.gif)
