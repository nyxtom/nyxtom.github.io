---
title: React useState and useEffect
description: Building on top of understanding react state management with the useEffect and properly managing cleanup.
tags: [javascript, react]
---

In a [previous article](/2021/07/05/react-state/) I mentioned a lot about how to setup [React](https://reactjs.org) state management and deal with concepts like [inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control). Let's just do a quick review of the implementation so far.

### Counters Data Service and State Emitter

We have the counters data service to store the state and respond to functions like `increment()`.

```javascript
// ./services/counters-data-service
import { StateEmitter } from './state-emitter';

export class CountersDataService extends StateEmitter {
    constructor() {
        super({ count: 1 })
    }

    increment() {
        this.state.count = this.state.count + 1;
    }
}
```

The *StateEmitter* is used to setup and handle publisher/subscribers and manages the Proxy for the state so we can handle whenever changes are made to the state.

```javascript
import { createProxy } from './proxy-util';

export class StateEmitter {
    constructor(state = {}) {
        this.listeners = new Map();
        this.state = this._createProxy(state, undefined, {
            set: (path, value) => {
                this.publish(path, value);
            }
        });
    }

    subscribe(key, listener) {
        if (typeof key === 'function' && typeof listener === 'undefined') {
            listener = key;
            key = '*';
        }

        if (!this.listeners.has(key)) {
            this.listeners.set(key, [listener]);
        } else {
            this.listeners.get(key).push(listener);
        }

        return {
            dispose: () => {
                let listeners = this.listeners.get(key);
                let idx = listeners.indexOf(listener);
                if (idx > -1) {
                    listeners.splice(idx, 1);
                }
            }
        };
    }

    publish(key, ...args) {
        if (key !== '*' && this.listeners.has(key)) {
            this.listeners.get(key).forEach(listener => {
                listener(...args);
            });
        }

        if (key !== '*' && this.listeners.has('*')) {
            this.listeners.get('*').forEach(listener => {
                listener(...args);
            });
        }
    }
}
```

Where in this case **createProxy** is a utility function that will iterate over the object to recursively create proxies whenever it encounters an object.

```javascript
function createProxy(value, path, options = {}) {
    let self = this;
    if (typeof value === 'object') {
        if (!Array.isArray(value)) {
            Object.keys(value).forEach(k => {
                let v = value[k];
                if (typeof v === 'object') {
                    let p = k;
                    if (path) {
                        p = `${path}.${p}`;
                    }
                    value[k] = createProxy(v, p, options);
                }
            });
        }
        return new Proxy(value, {
            get: function (target, name) {
                return target[name];
            },
            set: function (target, prop, v) {
                target[prop] = v;
                if (options.set) {
                    let p = prop;
                    if (path) {
                        p = `${path}.${p}`;
                    }
                    options.set(p, v);
                }
                return true;
            }
        });
    }

    return value;
}
```

### Counters Component

The **Counter** component will simply subscribe to the counters data service, and bind increment changes to the same service - which will propagate those changes. 

```javascript
import { Component } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export class Counter extends Component {
    constructor(props) {
        super(props);
        this.countersService = inject(config.services.Counters);
        this.onIncrement = () => this.countersService.increment();
        this.state = {
            count: 0,
            time: 0
        };
        this.subscriptions = [
            this.countersService.subscribe(this.setState.bind(this))
        ];
    }

    componentWillUnmount() {
        this.subscriptions.splice(0).forEach(sub => sub.dispose());
    }

    render() {
        return (
            <div>
                <h1>{this.state.count}: {this.state.time}</h1>
                <button onClick={this.onIncrement}>Click Me!</button>
            </div>
        );
    }
}
```

### Service Locator and Inject

The **inject** utility refers to the way we are using inversion of control and React's **useContext** to grab a **ServiceLocator** in order to resolve our dependency.

```javascript
import { createContext, useContext } from 'react';

export function inject(...services) {
    let locator = useContext(DepsContext);
    return locator.resolve(services);
}

export const DepsContext = createContext({});

export class ServiceLocator {
    constructor() {
        this.deps = new Map();
    }

    register(key, instance, options = {}) {
        if (typeof instance === 'undefined' && typeof key === 'function') {
            this.deps.set(key, { instance: key, options });
        } else {
            this.deps.set(key, { instance, options });
        }
    }

    resolve(deps) {
        if (Array.isArray(deps)) {
            return deps.map(d => this._lookup(d));
        } else if (deps && typeof deps === 'object') {
            return Object.keys(deps).reduce((acc, key) => {
                let name = deps[key];
                acc[name] = this._lookup(key);
                return acc;
            }, {});
        } else {
            return this._lookup(deps);
        }
    }

    _lookup(key) {
        let { instance, options } = this.deps.get(key);
        if (options.transient) {
            return new instance();
        }
        return instance;
    }
}
```

### App Setup and Registering Services

From the main app code, we just needed to setup the service locator and register the counters data service. The render code for the app will wrap the service locator as a **Context.Provider** when rendering so that the service locator will get bound into the context.

```javascript
import { render, createContext, Component } from 'react';

import { config } from './config';
import { DepsContext, ServiceLocator } from './utils/container';
import { CountersDataService } from './services/counters-data-service';

class App extends Component {
    constructor(props) {
        super(props);
        this.locator = new ServiceLocator();
        this.locator.register(config.services.Counters, new CountersDataService());
    }

    render() {
        return (
            <DepsContext.Provider value={this.locator}>
                <Counter />
            </DepsContext.Provider>
        );
    }
}

render(<App />, document.getElementById('root'));
```

## Pure Functions and useState

Now the above example component is a class component - this means all the normal React lifecycles will get executed as we would expect. If we were to create a functional component this might look different.

```javascript
import { useState } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = () => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    countersService.subscribe((state) => {
        setCount(state.count);
    });

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

To clarify, we should first determine what exactly **useState** is doing here. [useState](https://reactjs.org/docs/hooks-state.html), as part of the hooks api, provide us with the ability to add additional values that are stored for the component in an array. At initialization, the first item in the state array for the component is going to be the initial value. The initial value will be the value passed into the `useState(0)` function. When React calls our function, it's going to look at that cached index, the state array, and lookup the current value based on where the pointer is. Upon executing it will return both the current value based on the pointer and a setter dispatch function that will update the value in the store. That's pretty much it.

In the example above, however we are attempting to subscribe to the **countersService** right in the middle of the function. This might be fine for the first time this function is called, but every time the render function is called this is going to get executed. What would be better is to tie into the same lifecycle functions that the class component was using instead.

## useEffect

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = () => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        countersService.subscribe((state) => {
            setCount(state.count);
        });
    });

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

We are now using another hook api function called [useEffect](https://reactjs.org/docs/hooks-effect.html). **useEffect** combines the lifecycle functions **componentDidMount**, **componentDidUpdate**, and **componentWillUnmount**. It gives us the opportunity to perform additional operations, data fetching, manipulating the DOM after the fact, or perform "side effects". We're using it to subscribe to changes in the counters service. **useEffect** allows you to pass in two parameters; the first parameter is the function that will be executed on one of the lifecycle functions. The second parameter is a set of dependencies that tell react **when** we want to execute this function.

When you don't pass any second parameter then it's going to get executed after every render (first render and every update lifecycle function, as well as unmount). We can however customize this by passing an additional parameter of dependencies.

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = () => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        countersService.subscribe((state) => {
            setCount(state.count);
        });
    }, []);

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

Passing an empty array as in the above example will **only** run the function once on mount. We can customize this further but adding additional parameters to the array to only run the function whenever a particular value changes.

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = (props) => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        countersService.subscribe((state) => {
            setCount(state.count);
        });
    }, [props.id]);

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

This will effectively run the function on the initial first render and then subsequently **only** whenever the **props.id** changes. If you are familiar with **memoization** this is similar to that. We are effectively caching the output of the function based on the provided parameters.

One thing we haven't talked about in this example is cleanup. Our example will subscribe to the counters service changes but it does nothing to clean it up! React solves this by allowing you to return a cleanup function at the end of the effect.

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = (props) => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        let subscription = countersService.subscribe((state) => {
            setCount(state.count);
        });
        return () => {
            subscription.dispose();
        };
    });

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

In the above example now we are returning a clean up function. The second parameter to the **useEffect** as you will note is not including any array. This means that useEffect will be called on every render and update. The way this works with the clean up function is before the update is executed, similar to the useState cache, it will check against the current index and determine if it needs to execute the previous cleanup function before calling this function. This way you can subscribe/unsubscribe or **cleanup previous** then **setup** each time it is called. But what if we **only** wanted to run this once on **componentDidMount** and call the cleanup function when **componentWillUnmount**?

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = (props) => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        let subscription = countersService.subscribe((state) => {
            setCount(state.count);
        });
        return () => {
            subscription.dispose();
        };
    }, []);

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

That's it! Since we are passing an empty array the dependencies will never change - therefore it will only execute once and the cleanup function will be executed on **componentWillUnmount**. 

## Global State without useState

Since all we are doing is assigning the count, we can simplify this code by subscribing directly to the `count` and pass along the **setCount** function to the subscription.

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = (props) => {
    const [count, setCount] = useState(0);

    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        let subscription = countersService.subscribe('count', setCount)
        return () => {
            subscription.dispose();
        };
    }, []);

    return (
        <div>
            <h1>{count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

Now we don't necessarily **have** to use the `useState` function here. We could instead just render directly from the value in the **CountersDataService**.

```javascript
import { useState, useEffect } from 'react';

import { config } from '../config';
import { inject } from '../utils/container';

export const Counter = (props) => {
    const [invalidate, setInvalidate] = useState(false);
    const countersService = inject(config.services.Counters);
    const onIncrement = () => countersService.increment();

    useEffect(() => {
        let subscription = countersService.subscribe(() => setInvalidate(!invalidate));
        return () => {
            subscription.dispose();
        };
    }, []);

    return (
        <div>
            <h1>{countersService.state.count}</h1>
            <button onClick={onIncrement}>Click Me!</button>
        </div>
    );
};
```

In this case, we are making use of the **useState** function to effectively invalidate the state by passing in a new value `!invalidate` whenever the state changes within the counters service. There are several ways to do this from using the `useState`, `useReducer`, `useCallback` and others.

## Conclusion

There we go! Not much to it in terms of utilizing `useState` and `useEffect` hooks. We can setup our subscription code, and cleanup whenever the component is unmounting or cleanup the previous execution of an effect. We were able to learn about invalidating the component render with a simple invalidate state. While our **countersService.state.count** certainly works for our purposes since we are just displaying whatever current value is in that class - it unfortunately still has the pitfall that the underlying object is being mutated underneath us. Luckily the code is written in such a way that it will publish the changes - but there are edge cases that may still need to be caught. These are things that things like **useReducer** and the framework of [Redux](https://redux.js.org/) can help us with a more predictable state management system.

Thanks for following along!

Cheers
