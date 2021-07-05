---
title: React State and Inversion of Control
published: true
description: Working with React state, inversion of control, and Redux to understand the different ways to store and manipulate state in a React application.
tags: [javascript, react, redux]
---

When you're working on a web application inside a JavaScript framework, or even Vanilla JavaScript for that matter, there are generally different stages of state that you will run into. State refers to the actual data and manipulation of data as it ultimately is reflected in the view. When you write components, you will work with the properties of that component and the state of that view. Generally, the recommendation approach is to think about things in terms of necessity (i.e. reach for the right tool).

Within the context of React, there are different ways we can bind in data to our views.

### Local Variables

Setup a pure render function that uses just the given local scope to generate a view.

```javascript
export const App = () => {
    let name = 'Tom';

    return (
        <div>
            <h1>Hello {name}</h1>
        </div>
    );
};
```

### Properties

Setup a pure render function that uses a combination of local scope and properties.

```javascript
export const App = ({ name }) => {
    let date = new Date().toLocaleString();
    return (
        <div>
            <h1>Hello {name}</h1>
            <h2>Today is {date}</h2>
        </div>
    );
};
```

### State

Setup a render function that maintains some kind of state that can be changed based on user interactions or other manipulations in the application.

```javascript
import { useState } from 'react';

export const Counter = () => {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <h1>{count}</h1>
      <button onClick={() => setCount(v => v + 1)}>Click Me!</button>
    </div>
  );
};
```

In the above example we are taking advantage of the `useState` hook to provide us with a local state to our function. The equivalent in a class component could be written as.

```javascript
import { Component } from 'react';

export class Counter extends Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.increment = this.increment.bind(this);
  }
  
  increment() {
    this.setState((state) => {
      return { count: state.count + 1 };
    });
  }
  
  render() {
    return (
      <div>
        <h1>{this.state.count}</h1>
        <button onClick={this.increment}>Click Me!</button>
      </div>
    );
  }
}
```

A couple things about the above example component class. First, we are using the `constructor(props)` to initialize the state. This could be done with [class properties](https://babeljs.io/docs/en/babel-plugin-proposal-class-properties) by declaring the state at the class level. In addition to setting the state, we also need to **bind** the `this.increment.bind(this)` to ensure that **onClick** event handler calls to `Counter.prototype.increment` have the correct `this` scope. Sometimes this can be written as `this.onIncrement = this.increment.bind(this)` and the `onClick={this.onIncrement}` but this version works fine as well.

Now that we have a clean counter component, let's consider the scenario where we need some kind of global state.

## Global State

```javascript
import { Component } from 'react';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    super(props);
    this.countersService = Container.get(CountersDataService);
    
    this.onIncrement = () => this.countersService.increment();
  }
  
  render() {
    return (
      <div>
        <h1>{this.countersService.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

In our new example we are attempting to do a few things. One we want to get some kind of hypothetical `CountersDataService` from a `Container`. This utilizes a concept known as dependency injection to retrieve things via a Container. In Aurelia, for instance, we might use the `inject` function to perform this behavior. This could be re-written without the constructor like below:

```javascript
import { Component } from 'react';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  counterService = inject(CountersDataService);
  
  onIncrement = () => this.countersService.increment();
  
  render() {
    return (
      <div>
        <h1>{this.countersService.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

Let's assume that `inject` actually now refers to the same functionality as using `Container.get`. Hypothetical counters service aside, let's look at the problems with this code. Right now, we have no way of informing when state changes, in fact even if `inject(CountersDataService)` simply returned a new instance of that class and `.count` refers to just a plain value - we still have no way of informing react when state changes. 

> As an aside, using the class level properties in React like the above example will create new `Object.defineProperty` for every instance of the component. This differs from having properties on the prototype so you should be aware that this is happening.

In any case, we need a way such that when a call to `increment()` is made, we need `this.state` to be able to reflect that change with a call to `setState`. Or for that matter, any time any `.count` is changed.

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
      count: this.countersService.count
    };
  }
  
  render() {
    return (
      <div>
        <h1>{this.state.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

So here's an interesting prospect we have designed. With the new `inject` function we are now passing along the `this` to function. This should theoretically get us an instance of the `CountersDataService` and ensure that the listener is `this`. Whenever we call `this.countersService.increment` we would like that service to know about the listeners for the changed property `.count` and then trigger a set state on those listeners. How do we know about the listeners? Well, the first time we call `this.countersService.count` could be an getter defined property that will ensure the listener is subscribed. Unfortunately, we don't have a good of way to identify when we call `this.countersService.count`. By the time this getter is called, we may have needed to already connect those two together to make it work. Object.defineProperty for getters will not let us know of the caller so we need to find another way.

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
        counters: getState({ count: 1 }, this.countersService, this)
    };
  }
  
  render() {
    return (
      <div>
        <h1>{this.state.counters.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

The above implementation could modify the `countersService` (if it's not already done) to handle the getters and setters according to the `.count` property. Presumably getState takes some object of things we want to transform into properties we are interested in, followed by the place where it's retrieved, and finally the subscription. Unfortunately, now we have cluttered our code with some excess `getState` where it's probably not needed. We really want to leave things clean and just use state.

## Publisher-Subscriber Pattern

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
      count: this.countersService.count
    };
    this.countersService.subscribe((state) => {
        this.setState({ count: state.count });
    });
  }
  
  render() {
    return (
      <div>
        <h1>{this.state.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

One way around this is by simply using a simple `subscribe` pattern explicitly. This would rely upon the service implementing some kind of pub/sub pattern, but the gist is that the service doesn't have to care about `setState` or react at all. We could subscribe to just about anything in that service and update our local state accordingly to only what we need. This leaves the **React** code where it needs to be and the **Vanilla JS** code where *it* needs to be. Our example above however is a general **subscribe** to everything whenever the entire counters service object changes. We may wish to change this to a **specific** event or state change.

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
      count: this.countersService.count
    };
    this.countersService.subscribe('count', (count) => {
        this.setState({ count });
    });
  }
  
  render() {
    return (
      <div>
        <h1>{this.state.count}</h1>
        <button onClick={this.onIncrement}>Click Me!</button>
      </div>
    );
  }
}
```

For more general updates non-specific to a single property you may want to consider using a wrapped message of some kind. This will be different than the **catch all** subscription but it's a bit more than the **single property** events.

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService, CountersUpdate } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
        count: 0,
        time: 0
    };
    this.countersService.subscribe(CountersUpdate, (update) => {
        this.setState({ count: update.count, time: update.time });
    });
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

## Event Emitter and Observables

Let's say that in the case of the `CountersDataService` it is an **Event Emitter**, meaning that as changes are made to critical properties such as `count` it will emit those changes as events to anyone who cares. In the case of components that call `.subscribe` this can come in the form of a specific property name `count` or for catch all `*`, or maybe even more complex expressions where we only want to subscribe to events under certain conditions. What about in the case where we just want to basically pipe the data that the service happens to take care of and be able to access it directly from our component without having to call `setState`.

```javascript
import { Component } from 'react';
import { inject, observe } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
        count: 0,
        time: 0
    };
    /**
        // observe would presumably be just a helper function to a singleton instance 
        // of a PropertyObserver for instance or some kind of Observer.
        this.observer = new PropertyObserver();
        this.observer.observe(this.countersService, '*', this.setState.bind(this));
    */
    observe(this.countersService, '*', this.setState.bind(this));
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

In this above example, the code has been changed to essentially bind subscriptions directly to `this.setState`. This means that any changes that occur in the `countersService` according to the path we are interested in `*` will be passed along to `this.setState`. This kind of implementation is merely a minor revision of what we were already getting with the event emitter and the publisher/subscriber pattern except now it's just been wrapped into an `observe` utility function that will just use the same callback. 

The problem with using the `observe` function like in the above example is we have no way of cleaning things up when `componentWillUnmount` for instance. So despite making it more convenient, we've introduced a problem with leaking to `this.setState`.

```javascript
import { Component } from 'react';
import { inject, observe } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
    this.onIncrement = () => this.countersService.increment();
    this.state = {
        count: 0,
        time: 0
    };

    this.observer = new PropertyObserver();
    this.subscriptions = [
        this.observer.subscribe(this.countersService, '*', this.setState.bind(this))
    ];
  }

  componentWillUnmount() {
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

Great, we have our cleanup code now which will make sure to dispose of any subscriptions in the *PropertyObserver* that way when the component is unmounting we don't try and call `setState` again. What about in our previous example with the event emitter?

```javascript
import { Component } from 'react';
import { inject } from '../utils/container';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(CountersDataService);
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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

## Containers

Up until this point, we've sort of glossed over the `inject(CountersDataService)` functionality. The premise here is that we need **some** implementation of `CountersDataService` and we don't really care how it gets populated. **Containers** when they are setup can be setup in such a way where we can swap in and out certain types of implementations.

```javascript
import { Component } from 'react';
import { render } from 'react-dom';

import { config } from './config';
import { Container } from './utils/container';
import { Counter } from './components/counter';
import { CountersDataService } from './services/counters-data-service';

class App extends Component {
    constructor() {
        this.config = config;
        this.container = Container.setup();
        this.container.register(CountersDataService, new CountersDataService(config));
    }

    render() {
        return (
            <Counter />
        );
    }
}

render(<App/>, document.getElementById('root'));
```

We can easily swap out the base implementation with a more specific implementation that might deal with a mocked version for testing, or an offline version..etc

```javascript
import { Component } from 'react';
import { render } from 'react-dom';

import { config } from './config';
import { Container } from './utils/container';
import { Counter } from './components/counter';
import { CountersDataService } from './services/counters-data-service';
import { FakeCountersDataService } from './tests/mock/services/counters-data-service';

class App extends Component {
    constructor() {
        this.container = Container.setup();
        this.container.register(CountersDataService, new FakeCountersDataService());
    }

    render() {
        return (
            <Counter />
        );
    }
}

render(<App/>, document.getElementById('root'));
```

Alternatively, instead of registering the type `CountersDataService` we could use a string reference, an interface or whatever we are interested in doing.

```javascript
import { Component } from 'react';
import { render } from 'react-dom';

import { config } from './config';
import { Container } from './utils/container';
import { CountersDataService } from './services/counters-data-service';
import { Counter } from './components/counter';

class App extends Component {
    constructor() {
        this.container = Container.setup();
        this.container.register(config.services.Counters, new CountersDataService()); // config.services.Counters = 'counters'
    }

    render() {
        return (
            <Counter />
        );
    }
}

render(<App/>, document.getElementById('root'));
```

Then in the **Counter** component the container would inject via the `counters` string.

```javascript
import { Component } from 'react';

import { inject } from '../utils/container';
import { config } from '../config';
import { CountersDataService } from '../services/counters-data-service';

export class Counter extends Component {
  constructor(props) {
    this.countersService = inject(config.services.Counters); // or Container.get(config.services.Counters)
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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

## Service Locator

[Martin Fowler](https://martinfowler.com/articles/injection.html#UsingAServiceLocator) discusses the key benefit of a dependency injection is that **it removes the dependency that an Implementation has on the concrete implementation**. A service locator is then thought of as a singleton instance [Registry](https://martinfowler.com/eaaCatalog/registry.html). One way we can implement the idea of a service locator is through the use of the `useContext` in react. We don't necessarily need to leverage useContext for global state, but we *can* use it as a dependency injection or service locator.

```javascript
// /utils/container
import { createContext } from 'react';

//...

export const DepsContext = createContext({});
```

```javascript
import { render, createContext, Component } from 'react';

import { config } from './config';
import { DepsContext } from './utils/container';
import { ServiceLocator } from './utils/service-locator';
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

Great! Now we've wrapped the app in a context that houses our dependency context and attached in a hypothetical **ServiceLocator** instance. What about the **Counter**? How do we take advantage of the **DepsContext**?

```javascript
import { useContext, Component } from 'react';

import { config } from '../config';
import { DepsContext } from '../utils/container';

export class Counter extends Component {
  constructor(props) {
    super(props);

    /**
     * We can obtain the service locator because <DepsContext.Provider value={this.locator} was 
     * used in the app, so the result of calling useContext(DepsContext) will return the instance 
     * of the locator directly. We can now resolve dependencies directly now that we have the locator.
     */
    let locator = useContext(DepsContext);
    this.countersService = locator.resolve(config.services.Counters);
    // this.countersService = locator.resolve(CountersDataService); // alternatively resolve by type


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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

The above example will now use the service locator from just the React `useContext` function alone! In fact, technically you can just bypass the service locator entirely and just read directly off of the `DepsContext` alone. If we had created a `CountersDataServiceContext` for instance, we could use that class directly. Now all that needs to be done is to add a helper function which will take care of the `locator.resolve` for us.

```javascript
// /utils/container
import { createContext, useContext } from 'react';

export function inject(...services) {
    let locator = useContext(DepsContext);
    return locator.resolve(services);
}

export const DepsContext = createContext({});
```

Now the counter component can return to the original implementation with the use of `inject`.

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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

The implementation of the `ServiceLocator` then just needs to be a simple **Map**. We could expand on this **ServiceLocator** to provide more robust resolution options depending on how it is registered, but for now we will stick with a simple Map.

```javascript
export class ServiceLocator {
    constructor() {
        this.deps = new Map();
    }

    register(key, instance) {
        this.deps.set(key, instance);
    }

    resolve(deps) {
        if (Array.isArray(deps)) {
            return deps.map(d => this.deps.get(d));
        } else if (deps && typeof deps === 'object') {
            return Object.keys(deps).reduce((acc, key) => {
                let name = deps[key];
                acc[name] = this.deps.get(key);
                return acc;
            }, {});
        } else {
            return this.deps.get(deps);
        }
    }
}
```

Now, at the very least this will let us perform `locator.register('counters', new CountersDataService())` and `locator.resolve('counters')` but we may want to have the ability to register and resolve by a type rather than by a specific string. In our simple implementation we are using just the normal **Map** so we can pass along a function, class or a string and it will be assigned into the map. It would however, be nice if the way we registered the instance could handle whether the instance was supposed to be an actual instance or if we need to create a new one each time.

```javascript
import { CountersDataService } from './services/counters-data-service';

let locator = new ServiceLocator();
// register a **Type** as transient, meaning resolve will return a new instance each time
locator.register('counters', CountersDataService, { transient: true });
// register a singleton 
locator.register('counters', new CountersDataService());
// register just the actual **Type**, meaning resolve will return the Type (not an instance)
locator.register('counters', CountersDataService);
```

In order to provide a transient option, we need to update the register and resolve to take into account an `options`.

```javascript
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

## react-decoupler

Fortunately for us, all of this functionality is built into a small library called [react-decoupler](https://github.com/testdouble/react-decoupler). It uses the exact concepts discussed above from `useContext`, `createContext`, `ServiceLocator`, and the ability to resolve dependencies like we have been doing above. The implementation will change to:

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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

The **Counter** component won't change, we're going to update the `utils/container` to change how the `inject` function looks things up.

```javascript
// /utils/container
import { useLocator } from 'react-decoupler';

export function inject(...services) {
    let locator = useLocator();
    let result = locator.resolve(services);
    if (services.length === 1) {
        return result[0];
    } else {
        return result;
    }
}

export const DepsContext = createContext({});
```

Alternatively, we can use the helper function **useServices** from `react-decoupler` to inject the services directly without having to call our **inject** function.

```javascript
import { Component } from 'react';
import { useServices } from 'react-decoupler';

import { config } from '../config';

export class Counter extends Component {
  constructor(props) {
    super(props);
    let [ countersService ] = useServices([ config.services.Counters ]);
    this.countersService = countersService;
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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

Then in the **App**, we still need to register the context like we were doing before.

```javascript
import { render, Component } from 'react';
import { DecouplerProvider, ServiceLocator }

import { config } from './config';
import { CountersDataService } from './services/counters-data-service';

class App extends Component {
    constructor(props) {
        super(props);
        this.locator = new ServiceLocator();
        this.locator.register(config.services.Counters, new CountersDataService());
    }

    render() {
        return (
            <DecouplerProvider locator={this.locator}>
                <Counter />
            </DecouplerProvider>
        );
    }
}

render(<App />, document.getElementById('root'));
```

Alternatively, instead of creating a new **ServiceLocator** and calling **register** for each of the keys, we can directly pass in the map.

```javascript
import { render, Component } from 'react';
import { DecouplerProvider }

import { config } from './config';
import { CountersDataService } from './services/counters-data-service';

class App extends Component {
    constructor(props) {
        super(props);
        this.services = {
            [config.services.Counters]: new CountersDataService()
        };
    }

    render() {
        return (
            <DecouplerProvider services={this.services}>
                <Counter />
            </DecouplerProvider>
        );
    }
}

render(<App />, document.getElementById('root'));
```

There are additional options, as in the previous examples, to pass along options when you register such that you can either get the type, a new transient instance, or a singleton object like the above example.

```javascript
import { render, Component } from 'react';
import { DecouplerProvider, ServiceLocator }

import { config } from './config';
import { CountersDataService } from './services/counters-data-service';

class App extends Component {
    constructor(props) {
        super(props);
        this.locator = new ServiceLocator();
        this.locator.register(config.services.Counters, CountersDataService, {
            asInstance: true,
            // withParams: ['otherDependencies'], create instance with other deps
        });
    }

    render() {
        return (
            <DecouplerProvider locator={this.locator}>
                <Counter />
            </DecouplerProvider>
        );
    }
}

render(<App />, document.getElementById('root'));
```

## Event Emitters

Up until now, we haven't really provided any details about how to build an **EventEmitter** that can handle the publishers/subscribers. Note that in our previous examples we constructed a hypothetical *CountersDataService* that could be subscribed to.

```javascript
import { Component } from 'react';
import { useServices } from 'react-decoupler';

import { config } from '../config';

export class Counter extends Component {
  constructor(props) {
    super(props);
    let [ countersService ] = useServices([ config.services.Counters ]);
    this.countersService = countersService;
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
    this.subscriptions.forEach(sub => sub.dispose());
    this.subscriptions = [];
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

In the **CountersDataService** the current implementation is just the following:

```javascript
// /services/counters-data-service
export class CountersDataService extends EventEmitter {
    constructor() {
        super();
        this.state = {
            count: 0
        };
    }

    increment() {
        this.state.count++;
    }
}
```

Great, but this doesn't show us how to propogate changes, subscribe or publish whenever we change count. Let's extend the counters data service with an event emitter we will implement. **Note** this is different than the **EventEmitter** in Node.js.

The event emitter needs to support several features including:

- Subscribe to all changes with a listener callback
- Subscribe to a specific key
- Publish to a specific key
- Publish to all listener callbacks


```javascript
export class EventEmitter {
    constructor() {
        this.listeners = new Map();
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

This rudimentary implementation of the event listener gets the gist across. We subscribe to particular paths, push those onto the map and whenever publish is called we will also make sure to publish to the wildcard key as well. Now the **CountersDataService** just needs to be updated to also publish those changes.

```javascript
export class CountersDataService extends EventEmitter {
    constructor() {
        super();
        this.state = {
            count: 0
        };
    }

    increment() {
        this.state.count++;
        this.publish('count', value);
    }
}
```

So a few things we haven't considered in this solution. One, we **have** to call `publish`, now in most cases this is fine but we have to be careful that we don't forget to actually do it. In other cases, where we are propogating specific messages around this is fine, but for general state changes to random objects this will get annoying very fast. Instead, what would be nice is if we can take into account the fact that **subscribe** is telling us what path we are interested in.

```javascript
export class EventEmitter {
    constructor() {
        this.listeners = new Map();
        this._state = {};
    }

    subscribe(key, listener) {
        if (typeof key === 'function' && typeof listener === 'undefined') {
            listener = key;
            key = '*';
        }

        if (this.hasOwnProperty('state') && key !== '*') {
            let existingValue = this.state[key];
            this._state[key] = existingValue;
            let descriptor = Object.defineProperty(this.state, key, {
                get() {
                    return this._state[key];
                },
                set(val) {
                    this._state[key] = val;
                    this.publish(key, val);
                },
                enumerable: true,
                configurable: true
            })
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

    // ...
}
```

Great, this will at least attempt to override the existing state object with a brand new setter that will encapsulate the publish logic whenever `this.state.count = value` is assigned. But there's several problems with this approach. One, we have to have another internal state object to store things. Two, we aren't actually capturing wildcard changes - so if the only thing subscribing is a catch-all then this won't work.

## Proxy

With ES6 you can create something known as a [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) which enables you to intercept and redefine fundamental operations on a given object. You can intercept any number of operations from **get**, **set**, **construct**, **deleteProperty**, **apply**, **has**, **ownKeys**, **getPrototypeOf**, **setPrototypeOf**, **isExtensible**, **getOwnPropertyDescriptor**, **defineProperty**, **preventExtensions**. For our simple use case, we're going to create a **state** object proxy that will intercept gets and sets.

```javascript
export class StateEmitter {
    constructor(state = {}) {
        this.listeners = new Map();
        let self = this;
        this.state = new Proxy(state, {
            get: function (target, name) {
                return target[name];
            },
            set: function (target, prop, value) {
                target[prop] = value;
                self.publish(prop, value);
                return true;
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

Now in the data service, we can simply setup the count state like so.

```javascript
import { StateEmitter } from './state-emitter';

export class CountersDataService extends StateEmitter {
    
    constructor() {
        super({ count: 0 });
    }

    increment() {
        this.state.count = this.state.count + 1;
    }
}
```

Unfortunately, our contrived example while it does work for simple properties, the Proxy currently doesn't handle recursive intercepting whenever additional objects or even arrays are defined. Adding to that difficulty, we are now intercepting everything regardless of what is actually being subscribed to. Nevertheless, we do have a way to subscribe to **only** the properties we care about and update based on those changes as we did in previous samples. 

## Redux

There are some major flaws that stick out on our previous implementation of relying upon a state object proxy through the **StateEmitter** (even in its naive implementation) to manage state. One major issue is that in many cases where we are debugging an application sometimes it's useful to know the difference between what happened **before** something happened in the app, and given a change/interaction/update what happened after. 

Our implementation of the Proxy would need to make sure to handle setting those values to the state by wrapping the result in another Proxy (this making sure we have implemented some kind of recursive Proxy for these edge cases). This more robust implementation will certainly work for many cases but it still has a major flaw in that we are still mutating a single object.

Knowing the state over time is vital to debugging an application and when we are constantly mutating the same object it generally means we have to re-create the behavior and assign variance by sometimes mutating the local state manually while we are debugging. This can be error-prone and very time consuming. What if instead, we could actually just keep track of the state over time, even replay it? This is where [Redux](https://redux.js.org/) helps us **a lot**.

Redux combines the best of the ideas we've discussed above into one cohesive toolkit while adding more of its own useful features. Redux leverages the **Context API** similar to the way we were managing context and injecting services but it also takes care of a vital part of state management by making it immutable.

## Conclusion

Redux is something I'd like to go in more detail on another long post as this one is long enough already. We covered various levels of state management, the context api, inversion of control and dependency injection, and event emitters with the publish/subscriber model. We also covered the use of ES6 Proxies to intercept gets/sets in order to improve upon our publisher/subscriber model and update the UI. It's worth noting that Redux not only uses immutability - it does through the use of ES6 Proxies through the [Immer](https://immerjs.github.io/immer/complex-objects) library. In addition, Redux uses similar techniques with the Context API as we have talked about above when dealing with dependency injection for state management and services. We can make use of several of these strategies depending on our use case. As always, reach for the right tool. 

Cheers!
