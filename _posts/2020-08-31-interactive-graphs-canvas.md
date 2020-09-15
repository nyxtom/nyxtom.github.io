---
title: Drawing interactive graphs in canvas
published: true
description: How to use the canvas api and window events to create simple interactive edges and nodes in javascript
tags: [javascript,tutorial,webdev]
---

At my work we monitor network operations and infrastructure through a variety of tools like SNMP, NetFlow, Syslog...etc. One of the ways to help customers figure out what is going on in their networks is to visualize it through graphs! There are a number of great libraries to do this but the main one that I use quite often is [d3.js](https://d3js.org).

![D3](/assets/7krlehp97u076nu00lsu.png)

But this isn't a post about d3 (that's for another day), it's about utilizing [Canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Basic_usage) to draw stuff on the screen. More specifically, we want to draw a series of connected nodes in a graph and be able to drag these nodes around. Let's get started!

## Drawing nodes

First thing we will need to do is setup our canvas.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Map</title>
    <link rel="stylesheet" href="index.css">
    <script defer type="text/javascript" src="load.js"></script>
</head>
<body>
    <canvas></canvas>
</body>
</html>
```

```css
/** index.css */
:root {
    --root-font-size: 12px;
    --bg: #fafafa;
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

And now our javascript ⬇️ We're going to start off by keeping around an array of nodes that we want to draw. A node will consist of an **x**, **y**, **radius**, **fill**, **stroke**. These properties will correspond to canvas api methods when we go to draw them.

```javascript
const canvas = document.querySelector('canvas');
const context = canvas.getContext('2d');

var nodes = [];

function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}

window.onresize = resize;
resize();
```

Let's go ahead and add our `drawNode` function right now. We're going to use the [arc](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes#Arcs) function to draw at a point, radius and angles for the circle. We also manipulate the rendering context for the [fill, stroke](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Applying_styles_and_colors). Since we are generating a circle with the arc, we want the entire shape to be encapsulated in a [Path](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/beginPath) this is why we are using the **beginPath** function.

```javascript
function drawNode(node) {
    context.beginPath();
    context.fillStyle = node.fillStyle;
    context.arc(node.x, node.y, node.radius, 0, Math.PI * 2, true);
    context.strokeStyle = node.strokeStyle;
    context.stroke();
    context.fill();
}
```

## Mouse Functions

Since we want this to be interactive, let's add the ability to track when the user touches or clicks on the canvas and draw the node right there at the cursor position.

```javascript
function click(e) {
    let node = {
        x: e.x,
        y: e.y,
        radius: 10,
        fillStyle: '#22cccc',
        strokeStyle: '#009999'
    };
    nodes.push(node);
    drawNode(node);
}

window.onclick = click;
```

![Drawing Nodes](/assets/jpkhb0np6184hp0wm53q.png)

Great! Now we have some nodes drawn to the screen but we don't have any way to move them around. Let's take advantage of the target position on the **mouseDown** function so we can move things around with **mouseMove**.

```javascript
var selection = undefined;

function within(x, y) {
    return nodes.find(n => {
        return x > (n.x - n.radius) &&
            y > (n.y - n.radius) &&
            x < (n.x + n.radius) &&
            y < (n.y + n.radius);
    });
}

function move(e) {
    if (selection) {
        selection.x = e.x;
        selection.y = e.y;
        drawNode(selection);
    }
}

function down(e) {
    let target = within(e.x, e.y);
    if (target) {
        selection = target;
    }
}

function up(e) {
    selection = undefined;
}

window.onmousemove = move;
window.onmousedown = down;
window.onmouseup = up;
```

![Drawing Bug](/assets/5ao5vzeneeht2h9gfb7s.gif)

## Bug Fixes

#### Dragging causes the nodes to be rendered over and over

Uh oh! We need to fix this so we re-render all the nodes whenever this happens. To do this, we just need to add a bit of `clearRect` to the draw code and instead of `drawNode` we'll just call it **draw**.

```javascript
function click(e) {
    let node = {
        x: e.x,
        y: e.y,
        radius: 10,
        fillStyle: '#22cccc',
        strokeStyle: '#009999'
    };
    nodes.push(node);
    draw();
}

function move(e) {
    if (selection) {
        selection.x = e.x;
        selection.y = e.y;
        draw();
    }
}

function draw() {
    context.clearRect(0, 0, window.innerWidth, window.innerHeight);
    for (let i = 0; i < nodes.length; i++) {
        let node = nodes[i];
        context.beginPath();
        context.fillStyle = node.fillStyle;
        context.arc(node.x, node.y, node.radius, 0, Math.PI * 2, true);
        context.strokeStyle = node.strokeStyle;
        context.fill();
        context.stroke();
    }
}
```

#### Clicking and dragging can create a duplicate node

This works pretty well but the problem is if we click too quickly the nodes will appear when we mousedown and then move. Let's instead rely on the move event to clear the state when we want to create a new node.

We will get rid of the **window.onclick** and **click** code and instead rely on the `mousedown`, `mouseup`, `mousemove` events to handle *selection* vs *create* states. When the `mouseup` event occurs, if nothing is selected and it hasn't yet been moved then create a new node.

```javascript
/** remove the onclick code and update move and up code */
function move(e) {
    if (selection) {
        selection.x = e.x;
        selection.y = e.y;
        selection.moving = true;
        draw();
    }
}

function up(e) {
    if (!selection || !selection.moving) {
        let node = {
            x: e.x,
            y: e.y,
            radius: 10,
            fillStyle: '#22cccc',
            strokeStyle: '#009999',
            selectedFill: '#88aaaa'
        };
        nodes.push(node);
        draw();
    }
    if (selection) {
        delete selection.moving;
        delete selection.selected;
    }
    selection = undefined;
    draw();
}
```

![Selected Drag](/assets/xivnyui1r2oxix5duqtz.gif)

Great! Note, if you update the `draw` code to key off of the `selected` state you can change the fill like so:

```javascript
context.fillStyle = node.selected ? node.selectedFill : node.fillStyle;
```

## Adding Connections

The next thing we are going to do is at some edges to this graph. We want to be able to connect a line from one node to another. To do this, we're going to use a simple line for now and have an edges array defining these connections.

The behavior we want to accomplish is:

* **mousemove**, if there is a selection and the mouse is currently down ➡️ *update selection x and y*
* **mousedown**, find the node target, if there is a selection clear the selected state, then assign the selection to the target and set its selected state and draw
* **mouseup**, if there is no selection then create a new node and draw, otherwise if the current selection is not selected (because of mouse down) then clear the selection and draw after
* *additionally* **mousedown** when the selection changes to a new node and we have something already selected we can create an edge

```javascript
function move(e) {
    if (selection && e.buttons) {
        selection.x = e.x;
        selection.y = e.y;
        draw();
    }
}

function down(e) {
    let target = within(e.x, e.y);
    if (selection && selection.selected) {
        selection.selected = false;
    }
    if (target) {
        selection = target;
        selection.selected = true;
        draw();
    }
}

function up(e) {
    if (!selection) {
        let node = {
            x: e.x,
            y: e.y,
            radius: 10,
            fillStyle: '#22cccc',
            strokeStyle: '#009999',
            selectedFill: '#88aaaa',
            selected: false
        };
        nodes.push(node);
        draw();
    }
    if (selection && !selection.selected) {
        selection = undefined;
    }
    draw();
}
```

This is nearly the same result as before, except now we can control selection state. What I would like to happen is that we can add an edge such that the current selection and the new selection creates a new edge and line.

```javascript
var edges = [];

function draw() {
    context.clearRect(0, 0, window.innerWidth, window.innerHeight);

    for (let i = 0; i < edges.length; i++) {
        let fromNode = edges[i].from;
        let toNode = edges[i].to;
        context.beginPath();
        context.strokeStyle = fromNode.strokeStyle;
        context.moveTo(fromNode.x, fromNode.y);
        context.lineTo(toNode.x, toNode.y);
        context.stroke();
    }

    for (let i = 0; i < nodes.length; i++) {
        let node = nodes[i];
        context.beginPath();
        context.fillStyle = node.selected ? node.selectedFill : node.fillStyle;
        context.arc(node.x, node.y, node.radius, 0, Math.PI * 2, true);
        context.strokeStyle = node.strokeStyle;
        context.fill();
        context.stroke();
    }
}

function down(e) {
    let target = within(e.x, e.y);
    if (selection && selection.selected) {
        selection.selected = false;
    }
    if (target) {
        if (selection && selection !== target) {
            edges.push({ from: selection, to: target });
        }
        selection = target;
        selection.selected = true;
        draw();
    }
}
```

![Graph Nodes](/assets/uzo3lgvkj0esilypvq0d.gif)

That's it! Now we have some edges between nodes! In a follow-up to this post I will talk about Bezier Curves and how you can create some neat smooth interpolations between those curves what the Canvas api has to offer in terms of functionality here.
