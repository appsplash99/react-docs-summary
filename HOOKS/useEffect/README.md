<!DOCTYPE html>
<html lang="en">
<head>
  <style>
    body {
      width: 70vw;
      /* max-width: 650px; */
      margin: 0 auto;
    }
  </style>
</head>
<body>

# **Using the Effect Hook**

**The Effect Hook lets you perform side effects in function components:**

    - Data fetching, 
    - setting up a subscription,
    -  and manually changing the DOM in React components 
    are all examples of side effects.

**There are two common kinds of side effects in React components:**

    1. those that don’t require cleanup, 
    2. and those that do.
    
---

<br />


## **Effects Without Cleanup**
    Sometimes, we want to run some additional code after React has updated the DOM. 

    Examples of effects that don’t require a cleanup:
      - Network requests 
      - manual DOM mutations 
      - and logging are common.
        
    We say these are Effects without Side Effects because:
    we can run them and immediately forget about them.

**What does useEffect do?**

    By using this Hook, you tell React that your component needs to do something after render. 
    
    React will remember the function you passed 
    (we’ll refer to it as our “effect”), 
    and call it later after performing the DOM updates. 


**Does useEffect run after every render?**

    Yes! By default, 
    it runs both after the first render and after every update. 
    (We will later talk about how to customize this.)
    
    Instead of thinking in terms of “mounting” and “updating”, you might find it easier to think that effects happen “after render”. 
    
    React guarantees the DOM has been updated by the time it runs the effects.
---

<br >

## **Effects with Cleanup**

    Earlier, we looked at how to express side effects that don’t require any cleanup. However, some effects do.
    
    For example,
    we might want to set up a subscription to some external data source. 
    
    In that case,
    it is important to clean up so that we don’t introduce a memory leak! 
    

#### **Example Using Hooks**

    Let’s say we have a ChatAPI module that lets us subscribe to a friend’s online status. 

Here’s how we might subscribe and display that status:
    
```JSX
import React, { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  //-----------------------------------------------
  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });
  //-----------------------------------------------

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

**Why did we return a function from our effect?**

    This is the optional cleanup mechanism for effects. 

    Every effect may return a function that cleans up after it. 

    This lets us keep the logic for adding and removing subscriptions close to each other. 

    They’re part of the same effect!

**When exactly does React clean up an effect?** 

    React performs the cleanup when the component unmounts.

    However, as we learned earlier, 
    effects run for every render and not just once. 
    
    This is why React also cleans up effects from the previous render before running the effects next time. 
    
    We’ll discuss why this helps avoid bugs and how to opt out of this behavior in case it creates performance issues later below.
 
> **_NOTE:_**  
      We don’t have to return a named function from the effect. We called it ```cleanup``` here to clarify its purpose, but you could return an arrow function or call it something different.

---

<br />

## **Summary**

    We’ve learned that useEffect lets us express different kinds of side effects after a component renders. 

**Some effects might require cleanup so they return a function:**
```JSX
 useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });
```

**Other effects might not have a cleanup phase, and don’t return anything.**
```JSX
useEffect(() => {
    document.title = `You clicked ${count} times`;
  });
```
**The Effect Hook unifies both use cases with a single API.**

<br>

---
#
<br >

# **Tips for Using Effects**

> ## **Tip #1 : Use Multiple Effects to Separate Concerns**

Here is a component that combines the counter and the friend status indicator logic from the previous examples:

```JSX

class FriendStatusWithCounter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0, isOnline: null };
    this.handleStatusChange = this.handleStatusChange.bind(this);
  }

  componentDidMount() {
    document.title = `You clicked ${this.state.count} times`;
    ChatAPI.subscribeToFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  componentDidUpdate() {
    document.title = `You clicked ${this.state.count} times`;
  }

  componentWillUnmount() {
    ChatAPI.unsubscribeFromFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  handleStatusChange(status) {
    this.setState({
      isOnline: status.isOnline
    });
  }
  // ...

```

Note how the logic that sets ```document.title``` is split between ```componentDidMount``` and ```componentDidUpdate```. 
The subscription logic is also spread between ```componentDidMount``` and ```componentWillUnmount```.
And ```componentDidMount``` contains code for both tasks.


**So, how can Hooks solve this problem?**

Just like [you can use the State Hook more than once](https://reactjs.org/docs/hooks-state.html#tip-using-multiple-state-variables), 
you can also use several effects. 

This lets us separate unrelated logic into different effects:

```JSX

function FriendStatusWithCounter(props) {
  const [count, setCount] = useState(0);

  //************************************************
  useEffect(() => { // first useEffect
  document.title = `You clicked ${count} times`;
  });
  

  const [isOnline, setIsOnline] = useState(null);

  //************************************************
  useEffect(() => { // second useEffect
  function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });
  
  // ...
}
```

Hooks let us split the code based on what it is doing rather than a lifecycle method name.

> **Note**
React will apply every effect used by the component, in the order they were specified.


## **Explanation: Why Effects Run on Each Update**

Consider this [class component example](https://reactjs.org/docs/hooks-effect.html#example-using-classes-1)  with ```FriendStatus``` component that displays whether a friend is online or not. Our class reads ```friend.id``` from ```this.props```, subscribes to the friend status after the component mounts, and unsubscribes during unmounting:

``` JSX 
  componentDidMount() {
    ChatAPI.subscribeToFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  componentWillUnmount() {
    ChatAPI.unsubscribeFromFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }
```

**But what happens if the friend prop changes** while the component is on the screen? \
Our component would continue displaying the online status of a different friend. \
This is a bug. \
We would also cause a memory leak or crash when unmounting since the unsubscribe call would use the wrong friend ID.

*In a class component, we would need to add componentDidUpdate to handle this case:*

```JSX 
componentDidMount() {
    ChatAPI.subscribeToFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }

  //---HANDELING NEW FRIEND PROP CHANGE BELOW---------
  componentDidUpdate(prevProps) {
    // Unsubscribe from the previous friend.id
    ChatAPI.unsubscribeFromFriendStatus(
      prevProps.friend.id,
      this.handleStatusChange
    );
    // Subscribe to the next friend.id
    ChatAPI.subscribeToFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }
  //---HANDELING NEW FRIEND PROP CHANGE ABOVE---------

  componentWillUnmount() {
    ChatAPI.unsubscribeFromFriendStatus(
      this.props.friend.id,
      this.handleStatusChange
    );
  }
```



*Now consider the version of this component that uses Hooks:*

```JSX
function FriendStatus(props) {
  // ...
  useEffect(() => {
    // ...
    ChatAPI.subscribeToFriendStatus(
      props.friend.id,
      handleStatusChange
    );

    return () => { 
      // cleanup function 
      // RUNS ONLY WHEN COMPONENT UNMOUNTS
      
      ChatAPI.unsubscribeFromFriendStatus(
        props.friend.id,
        handleStatusChange
      );
    };
  });
```

> **Note** \
  There is no special code for handling updates because ```useEffect``` handles them by *default*.
  \
  ```useEffect``` cleans up the previous effects before applying the next effects.


*To illustrate the above,\
here is a sequence of subscribe and unsubscribe calls that this component could produce over time:*
```JSX

// Mount with { friend: { id: 100 } } props
ChatAPI.subscribeToFriendStatus(100, handleStatusChange);     // Run first effect

// Update with { friend: { id: 200 } } props
ChatAPI.unsubscribeFromFriendStatus(100, handleStatusChange); // Clean up previous effect
ChatAPI.subscribeToFriendStatus(200, handleStatusChange);     // Run next effect

// Update with { friend: { id: 300 } } props
ChatAPI.unsubscribeFromFriendStatus(200, handleStatusChange); // Clean up previous effect
ChatAPI.subscribeToFriendStatus(300, handleStatusChange);     // Run next effect

// Unmount
ChatAPI.unsubscribeFromFriendStatus(300, handleStatusChange); // Clean up last effect

```
#
<br>

> ## **Tip #2: Optimizing Performance by Skipping Effects**

In some cases, cleaning up or applying the effect after every render might create a performance problem.

*In class components, we can solve this by writing an extra comparison with prevProps or prevState inside componentDidUpdate:*

```JSX 
componentDidUpdate(prevProps, prevState) {
  if (prevState.count !== this.state.count) {
    document.title = `You clicked ${this.state.count} times`;
  }
}
```

*This requirement is common enough that it is built into the ```useEffect``` Hook API.\
 You can tell React to skip applying an effect if certain values haven’t changed between re-renders. \
 To do so, pass an array as an optional second argument to ```useEffect```:*

```JSX 
useEffect(() => {
  document.title = `You clicked ${count} times`;
}, [count]); // Only re-run the effect if count changes
```

*In the example above, we pass [```count```] as the second argument.* 

**What does this mean?**

If the ```count``` is ```5```, and then our component re-renders with ```count``` still equal to ```5```, \
React will compare ```[5]``` from the previous render and ```[5]``` from the next render. \
Because all items in the array are the same ```(5 === 5)```, \
React would skip the effect. That’s our optimization.

When we render with ```count``` updated to ```6```,\
React will compare the items in the ```[5]``` array, \
from the previous render to items in the ```[6]``` array from the next render. \
This time, React will re-apply the effect because ```(5 !== 6)```. \
If there are multiple items in the array, \
React will re-run the effect even if just one of them is different.

*This also works for effects that have a cleanup phase:*
```JSX
useEffect(() => {
  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
  return () => {
    ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
  };
}, [props.friend.id]); // Only re-subscribe if props.friend.id changes
```












</body>
</html>
