<img src="https://user-images.githubusercontent.com/1355779/61337603-f9fffd80-a7ea-11e9-9cb3-fa82e044c86e.png" alt="Resourcerer Icon" height="200" width="200" />

# resourcerer

`resourcerer` is a library for declaratively fetching and caching your application's data. Its powerful [`useResources`](#useresources) React hook or [`withResources`](#withresources) higher-order React component (HOC) allows you to easily construct a component's data flow, including:

* serial requests
* prioritized rendering for critical data (enabling less critical or slower requests to not block interactivity)
* delayed requests
* prefetching
* inter-model syncing
* ...and more

Additional features include:

* fully declarative (no more writing any imperative Fetch API calls)
* first-class loading and error state support
* smart client-side caching
* lazy fetching
* refetching
* forced cache invalidation
* updating a component when a resource updates
* zero dependencies
* under 10kB gzipped!

Getting started is easy:

1. Define a model in your application (these are classes descending from [Model](/docs/model.md) or [Collection](/docs/collection.md)):

```js
// js/models/todos-collection.js
import {Collection} from 'resourcerer';

export default class TodosCollection extends Collection {
  url() {
    return '/todos';
  }
}
```

2. Create a config file in your application and add your constructor to the ModelMap with a key:

```js
// js/core/resourcerer-config.js
import {register} from 'resourcerer';
import TodosCollection from 'js/models/todos-collection';

// choose any string as its key, which becomes its ResourceKey
register({todos: TodosCollection});
```

```js
// in your top level js file
import 'js/core/resourcerer-config';
```

3. Use your preferred abstraction (`useResources` hook or `withResources` HOC) to request your models in any component:

    1. ### useResources
    
        ```jsx
        import {useResources} from 'resourcerer';

        // tell resourcerer which resource you want to fetch in your component
        const getResources = (props) => ({todos: {}});
        
        function MyComponent(props) {
          const {
            isLoading,
            hasErrored,
            hasLoaded,
            todosCollection
          } = useResources(getResources, props);
          
          // when MyComponent is mounted, the todosCollection is fetched and available
          // as `todosCollection`!
          return (
            <div className='MyComponent'>
              {isLoading ? <Loader /> : null}
        
              {hasErrored ? <ErrorMessage /> : null}
       
              {hasLoaded ? (
                <ul>
                  {todosCollection.toJSON().map(({id, name}) => (
                    <li key={id}>{name}</li>
                  ))}
                </ul>
              ) : null}
            </div>
          );
        }
        ```
    
    1. ### withResources
    
        ```jsx
        import React from 'react';
        import {withResources} from 'resourcerer';

        // tell resourcerer which resource you want to fetch in your component
        @withResources((props) => ({todos: {}}))
        class MyComponent extends React.Component {
          render() {
            // when MyComponent is mounted, the todosCollection is fetched and available
            // as `this.props.todosCollection`!
            return (
              <div className='MyComponent'>
                {this.props.isLoading ? <Loader /> : null}
        
                {this.props.hasErrored ? <ErrorMessage /> : null}
        
                {this.props.hasLoaded ? (
                  <ul>
                    {this.props.todosCollection.map((todoModel) => (
                      <li key={todoModel.id}>{todoModel.get('name')}</li>
                    ))}
                  </ul>
                ) : null}
              </div>
            );
          }
        }
        ```

There's a lot there, so let's unpack that a bit. There's also a lot more that we can do there, so let's also get into that. But first, some logistics:

# Contents  

1. [Installation](#installation)
1. [Nomenclature](#nomenclature)
1. [Tutorial](#tutorial)
    1. [Intro](#tutorial)
    1. [Other Props Returned from the Hook/Passed from the HOC (Loading States)](#other-props-returned-from-the-hookpassed-from-the-hoc-loading-states)
    1. [Requesting Prop-driven Data](#requesting-prop-driven-data)
    1. [Changing Props](#changing-props)
    1. [Common Resource Config Options](#common-resource-config-options)
        1. [params](#params)
        1. [path](#path)
        1. [noncritical](#noncritical)
        1. [force](#force)
        1. [Custom Resource Names](#custom-resource-names)
        1. [prefetches](#prefetches)
        1. [data](#data)
        1. [lazy](#lazy)
        1. [minDuration](#minduration) 
        1. [fetch](#fetch)
        1. [dependsOn](#dependson)
        1. [provides](#provides)
    1. [Data mutations](#data-mutations)
    1. [Serial Requests](#serial-requests)
    1. [Canonical Models](#canonical-models)
    1. [Differences between useResources and withResources](#differences-between-useresources-and-withresources)
    1. [Using resourcerer with TypeScript](docs/typescript.md)
    1. [Caching Resources with ModelCache](#caching-resources-with-modelcache)
    1. [Declarative Cache Keys](#declarative-cache-keys)
    1. [Prefetch on Hover](#prefetch-on-hover)
    1. [Refetching](#refetching)
    1. [Cache Invalidation](#cache-invalidation)
    1. [Tracking Request Times](#tracking-request-times)
1. [Configuring resourcerer](#configuring-resourcerer)
1. [FAQs](#faqs)
1. [Migrating to v2.0](#migrating-to-v20)


# Installation

`$ npm i resourcerer` or `yarn add resourcerer`

`resourcerer` requires React >= 16.14 (peer range includes React 17, 18, and 19) but has no external dependencies.

Note: Resourcerer is written in TypeScript and is compiled to ESNext. It does no further transpiling&mdash;including `import`/`export`.
If you are using TypeScript yourself, this won't be a problem. If you're not, and you're not babelifying (or similar) your `node_modules` folder, you'll need to make an exception for this package, ie:

```js
// webpack.config.js or similar
module: {
  rules: [{
    test: /\.jsx?$/,
    exclude: /node_modules\/(?!(resourcerer))/,
    use: {loader: 'babel-loader?cacheDirectory=true'}
  }]
}
```

# Nomenclature

1. **Props**. Going forward in this tutorial, we'll try to describe behavior of both the `useResources` hook and the `withResources` HOC at once; we'll also rotate between the two in examples. Note that if we talking about a passed prop of, for example `isLoading`, that that corresponds to an `isLoading` property returned from the hook and a `this.props.isLoading` prop passed down from the HOC. 
  
1. **ResourceKeys**. These are the keys of the object passed to the `register` function in your top-level `resourcerer-config.js` file (discussed above in the introduction). The object is of type `Record<ResourceKeys, new () => Model | new () => Collection>`. These keys are passed to the executor functions and are used to tell the hook or HOC which resources to request.

1. **Executor Function**. The executor function is a function that both the hook and HOC accept that declaratively describes which resources to request and with what config options. In these docs you'll often see it assigned to a variable called `getResources`. It accepts `props` as arguments and may look like, as we'll explore in an example later:

    ```js
    const getResources = (props) => ({user: {path: {userId: props.id}}});
    ```

    or

    ```js
    const getResources = (props) => {
      const now = Date.now();
      
      return {
        userTodos: {
          params: {
            limit: 20,
            end_time: now,
            start_time: now - props.timeRange,
            sort_field: props.sortField
          }
        }
      };
    };
    ```

    It returns an object whose keys represent the resources to fetch and whose values are **Resource Configuration Objects** that we'll discuss later (and is highlighted below).
    
1. **Resource Configuration Object**. In the object returned by our executor function, each entry has a key equal to one of the `ResourceKeys` and whose value we will refer to in this document as a Resource Configuration Object, or Resource Config for short. It holds the declarative instructions that `useResources` and `withResources` will use to request the resource.

# Tutorial

Okay, back to the initial example. Let's take a look at our `useResources` usage in the component:

```js
// `@withResources((props) => ({todos: {}}))`
const getResources = (props) => ({todos: {}});

export default function MyComponent(props) {
  const resources = useResources(getResources, props);
  
  // ...
}
```

You see that `useResources` takes an executor function that returns an object. The executor function
takes a single argument: the current props, which are component props when you use `withResources`, but can be anything when you use `useResources`. The executor function returns an object whose keys are `ResourceKeys` and whose values are Resource Config objects. Where do `ResourceKeys` come from? From the object passed to the `register` method in the config file we added earlier!

```js
// js/core/resourcerer-config.js
import {register} from 'resourcerer';
import TodosCollection from 'js/models/todos-collection';

// after adding this key, `todos` can be used in our executor functions to reference the Todos resource.
// The 'todos' string value will also be the default prefix for all todos-related return values.
// That's why we have `props.todosCollection`!
register({todos: TodosCollection});
```

(We can also pass custom prefixes for our prop names in a component, but [we'll get to that later](#custom-resource-names).)  

Back to the executor function. In the example above, you see it returns an object of `{todos: {}}`. In general, the object it should return is of type `{[key: ResourceKeys]: ResourceConfigObject}`, where `ResourceConfigObject` is a generic map of config options. It can contain as many keys as resources you would like the component to request. In our initial example, the Resource Config Object was empty. Further down, we'll go over the plethora of options and how to use them. For now, let's take a look at some of the resource-related props this simple configuration provides our component.


## Other Props Returned from the Hook/Passed from the HOC (Loading States)

Of course, in our initial example, the `todosCollection` won’t be populated with data immediately since, after all, the resource has to be fetched from the API.  Some of the most **significant** and most common React UI states we utilize are whether a component’s critical resources have loaded entirely, whether any are still loading, or whether any have errored out. This is how we can appropriately cover our bases&mdash;i.e., we can ensure the component shows a loader while the resource is still in route, or if something goes wrong, we can ensure the component will still fail gracefully and not break the layout. To address these concerns, the `useResources` hook/`withResources` HOC gives you several loading state helper props. From our last example:


- `todosLoadingState` (can be equal to any of the [LoadingStates constants](https://github.com/noahgrant/resourcerer/blob/main/lib/types.ts#L4). There will be one for each resource, and the property names will be equal to `${resourceKey}LoadingState`)
- `todosStatus` {number} - the HTTP status code from the most recent response for that resource (one per resource as `${resourceKey}Status`)
- `hasLoaded` {boolean} - all critical resources have successfully completed and are ready to be used by the component
- `isLoading` {boolean} - any of the critical resources are still in the process of being fetched
- `hasErrored` {boolean} - any of the critical resource requests did not complete successfully
  
`isLoading` , `hasLoaded` , and `hasErrored` are not based on individual loading states, but are rather a collective loading state for the aforementioned-critical component resources. In the previous example, the todos resource is the only critical resource, so `isLoading` / `hasLoaded` / `hasErrored` are solely based on `todosLoadingState`. But we can also add a non-critical `users` resource, responsible, say, for only display users' names alongside their TODOs&mdash;a small piece of the overall component and not worth delaying render over. Here’s how we do that:

```js
const getResources = (props) => ({
  todos: {},
  users: {noncritical: true}
});

function MyClassWithTodosAndAUsers(props) {
  const resources = useResources(getResources, props);
}
```

`MyClassWithTodosAndAUsers` will now receive the following loading-related props, assuming we've registered a `usersCollection` in our config file:

- `todosLoadingState`
- `usersLoadingState` 
- `isLoading`
- `hasLoaded` 
- `hasErrored`

In this case, `isLoading` , et al, are only representative of `todosLoadingState` and completely irrespective of `usersLoadingState` . This allow us an incredible amount of flexibility for rendering a component as quickly as possible.

Here’s how might use that to our advantage in `MyClassWithTodosAndAUsers` :

```jsx
import {Utils} from 'resourcerer';

function MyClassWithTodosAndAUsers(props) {
  const {
    isLoading,
    hasErrored,
    hasLoaded,
    todosCollection,
    usersCollection,
    usersLoadingState
  } = useResources(getResources, props);
  
  var getUserName = (userId) => {
    // usersCollection guaranteed to have returned here
    var user = usersCollection.find(({id}) => id === userId);
            
    return (
      <span className='user-name'>
        {user && user.id || 'N/A'}
      </span>
    );
  };
          
  return (
    <div className='MyClassWithTodosAndUsers'>
      {isLoading ? <Loader /> : null}
          
      {hasLoaded ? (
        // at this point we are guaranteed all critical resources have returned.
        // before that, todosCollection is still a Collection instance that can be
        // mapped over--it's just empty
        <ul>
          {todosCollection.map((todoModel) => (
            <li key={todoModel.id}>
              // pure function that accepts loading states as arguments
              {Utils.hasLoaded(usersLoadingState) ?
                getUserName(todoModel.get('userId')) :
                // if you're anti-loader, you could opt to render nothing and have the
                // user name simply appear in place after loading
                <Loader type="inline" />}
              {todoModel.get('name')}
            </li>
          )}
        </ul>
      ) : null}
          
      {hasErrored ? <ErrorMessage /> : null}
    </div>
  );
```

Here's a real-life example from the [Sift](https://sift.com) Console, where we load a customer's workflows without waiting for the workflow stats resource, which takes much longer. Instead, we gracefully show small loaders where the stats will eventually display, all-the-while keeping our console interactive:

![Noncritical Resource Loading](https://user-images.githubusercontent.com/1355779/57596645-99a9c280-7500-11e9-916d-f60cfd00ee10.png)

And here's what it looks like when the stats endpoint returns:

![Noncritical Resource Returned](https://user-images.githubusercontent.com/1355779/57596646-9a425900-7500-11e9-8121-5ced72c0fcba.png)

There’s one other loading prop offered from the hook/HOC: `hasInitiallyLoaded`. This can be useful for showing a different UI for components that have already fetched the resource. An example might be a component with filters: when a filter is changed after the initial resource is loaded (thus re-fetching the resource), we may want to show a loader with an overlay over the previous version of the component. See the [Advanced Topics docs](/docs/advanced_topics.md#loading-overlays) for more.


## Requesting Prop-driven Data

Let's say we wanted to request not the entire users collection, but just a specific user. Here's our config:

```js
// js/core/resourcerer-config.js
import {register} from 'resourcerer';
import TodosCollection from 'js/models/todos-collection';
import UserModel from 'js/models/user-model';

register({
  todos: TodosCollection,
  user: UserModel
});
```

And here's what our model might look like (NOTE: use a Model when you've got one resource instance, and a Collection when you've got a list of that resource):

```js
// js/models/user-model.js
import {Model} from 'resourcerer';

export default class UserModel extends Model {
  url({userId}) {
    // `userId` is passed in here because we have
    // `path: {userId: props.id}` in the resource config object
    return `/users/${userId}`;
  }
  
  static dependencies = ['userId'];
}
```

The `dependencies` static property is important here, as we'll see in a second; it is a list of properties that `resourcerer` will use to generate a cache key for the model. It will look for the `userId` property in the following places, in order:

1. the `path` object
1. the `data` object
1. the `params` object

All three of these come from via the [Resource Configuration Object](#nomenclature) that is returned from our executor function; it might look like this:

```jsx
const getResources = (props) => ({user: {path: {userId: props.id}}}) 

// hook
function MyComponent(props) {
  const resources = useResources(getResources, props);
  
  // ...
}

// HOC
@withResources(getResources)
class MyComponentWithAUser extends React.Component {}
```
 
Assuming we have a `props.id` equal to `'noahgrant'`, this setup will put `MyComponentWithAUser` in a loading state until `/users/noahgrant` has returned.
  
### *...and here's the best part:*
  
Let's say that `props.id` changes to a different user. `MyComponentWithAUser` will get put _back_ into a loading state while the new endpoint is fetched, _without us having to do anything!_ This works because our model has dictated that its models should be cached by a `userId` field, which is passed to it in the [`path` property](#path).

## Changing Props
In general, there are two ways to change `props.id` as in the previous example:

1. Change the url, which is the top-most state-carrying entity of any application. The url can be changed either by path parameter or query paramter, i.e. `example.com/users/noahgrant` -> `example.com/users/bobdonut`, or `example.com/users?id=noahgrant` -> `example.com/users?id=bobdonut`. In this case, each prop change is _indexable_, which is sometimes desirable, sometimes not.

1. Change internal application state. For these cases, `useResources`/`withResources` make available another handy prop: `setResourceState`. `setResourceState` is a function that has the same method signature as the `useState` we all know and love. It sets internal hook/HOC state, which is then returned/passed down, respectively, overriding any initial prop, ie `setResourceState((state) => ({...state, id: 'bobdonut'}))`. This is _not_ indexable.

    Note that `setResourceState` is very useful for the `withResources` HOC because it allows you to 'lift' state above the fetching component that otherwise would not be possible. For `useResources`, it is a nice-to-have in some cases, but because you can always define your own `useState` above the `useResources` invocation, you may find that you use it less often.



## Common Resource Config Options

### params

The `params` option is passed directly to the [sync method](/docs/model.md#sync) and sent either as stringified query params (GET requests) or as a body (POST/PUT). Its properties are also referenced when generating a cache key if they are listed in a model's static `dependencies` property (See the [cache key section](#declarative-cache-keys) for more). Let's imagine that we have a lot of users and a lot of todos per user. So many that we only want to f