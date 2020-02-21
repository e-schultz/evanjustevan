---
template: post
title: Improving your mental model of useEffect
slug: improving-your-mental-model-of-useeffect
draft: false
date: 2019-08-20T16:49:00.000Z
description: Improving your mental model of useEffect
socialImage: /media/useeffect_desktop_16_9.png
category: Development
tags:
  - react
---
![gears turning ](/media/useeffect_desktop_16_9.png "Improving your mental model of useEffect")

Hooks landed in React a few months ago, and there has been a lot of excitement around them in terms of figuring out how to best use them, best practices, and how they map to current concepts in React and the lifecycle.

Many React developers are familiar with the React Component lifeCycle, and hooks like:

* [componentDidMount](https://reactjs.org/docs/react-component.html#componentdidmount)
* [componentDidUpdate](https://reactjs.org/docs/react-component.html#componentdidupdate)
* [shouldComponentUpdate](https://reactjs.org/docs/react-component.html#shouldcomponentupdate)

etc.

When trying to understand the `useEffect` hook, it's natural to want to map it to the lifecycle methods we already know. At first glance, `useEffect` seems to be like a combination of `componentDidMount` and `componentDidUpdate`.

While this can be a useful way of looking at it at first, it may not be the most accurate.

Instead of thinking in terms of 'what do I want to do when I mount, or when I update', it's more useful to ask:

> What values does this effect depend on?

To better understand where the idea of `useEffect = componentDidMount + componentDidUpdate` comes from, we will first look at a typical class-based component that is doing some data fetching.

```jsx
export default SearchComponent extends Component {
  constructor(props) {
    super(props);
    this.state = {
      results: []
    }
  }
  componentDidMount() {
    this.query(this.props.id)
  }
  componentDidUpdate(prevProps) {
    if(this.prevProps.id !== this.props.id) {
      this.query(this.props.id);
    }
  }
  query(id) {
    this.setState({isLoading: true})
    fetch(`/some/url/${id}`)
      .then(r=>r.json())
      .then(r=>this.setState({
        results: results
      });
    )
  }
}
```

When the component first mounts, we fetch data for the id that has been passed down as a prop. When the component updates, many things other than the id prop changing can cause this method to run, so we want to ensure that id has actually changed - or some poor server is going to get a DDoS attack with a bunch of API calls that we don't need.

While the lifecycle hooks of `componentDidMount` and `componentDidUpdate` with class-based components are common places to make a request based on a property, the fact that the component is mounting or updating is not really the thing we are concerned with.

## What we are actually concerned with?

> "what data does the query depend on?"

Before looking at how to handle this with `useEffect`, lets quickly review the [API of useEffect](https://reactjs.org/docs/hooks-reference.html#useeffect):

* Accepts a function
* If it returns a function, it will do cleanup when the component is unmounted
* Has an optional 2nd argument to pass in the data it depends on

One of the key things to keep in mind is the importance of that second argument, the [React Docs](https://reactjs.org/docs/hooks-effect.html) go into this in detail, but a summary is:

* If we leave it blank - it will run on every single render.
* If we pass in an empty array - it will execute only when the component mounts, and not on any updates
* If we pass in a value - it will execute when any of those values change
* If you are using the [react-hooks eslint plugin](https://www.npmjs.com/package/eslint-plugin-react-hooks) (and you should) - not providing the dependencies to your useEffect will give you warnings.

```jsx
export default SomeComponent = ({id}) => {
  let [results, setResults] = useState([]);
  useEffect(()=>{
    fetch(`/some/url/${id}`)
      .then(r=>r.json())
      .then(r=>setResults(r))
  },[id])
}
```

In the class-based version, making API calls feels very imperative - when this method is called, I want to check if/how a value has changed, and if it has changed - I want to call a method.

If the component is being created or updated often isn't the thing the matters. What we actually care about is "has the values that I care about changed?".

Before hooks were introduced, `componentDidMount` and `componentDidUpdate` were the best tools for the job at the time. With the hook based version, we are able to express this intent in a more declarative way: "I want to fetch data when the id changes"

# How do we identify what the effect depends on?

The eslint plugin can guide you in the right direction, but the short version is: "is there a variable that impacts how we run the effect?" If so, add it to the dependencies.

To demonstrate this, let's add an extra query parameter to our search:

```jsx
export default SomeComponent = ({id, filter}) => {
  let [results, setResults] = useState([]);

  useEffect(()=>{
    fetch(`/some/url/${id}?filter=${filter}`)
      .then(r=>r.json())
      .then(r=>setResults(r))
  },[id])
}
```

Even though we have added filter to the fetch query string, we have not added it to the dependencies of `useEffect`. 

As we update the filter, we won't be calling the API on any of the other updates, and it will only run when the id has changed.

Fixing this can be simple enough - add the filter to the list of dependencies for the `useEffect`.

```jsx
export default SomeComponent = ({id, filter}) => {
  let [results, setResults] = useState([]);

  useEffect(()=>{
    fetch(`/some/url/${id}?filter=${filter}`)
      .then(r=>r.json())
      .then(r=>setResults(r))
  },[id, filter])
}
```

As you can see, to properly use `useEffect`, in this case, we don't care if the component is mounting, or updating, or where it is in the life cycle. 

What we do care about is what data does this effect depend on.

`useEffect` is a very useful tool to add to our toolbox when working with React, but it can also be one of the more difficult hooks to fully understand.

Hopefully, this post can help clarify things a little bit better, but if you are curious for a deeper dive, be sure to check out Dan Abramovs' post, [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)

- - -

*this article was initially posted on the [rangle.io blog](https://rangle.io/blog/improving-your-mental-model-of-useeffect) and [medium](https://medium.com/rangle-io/improving-your-mental-model-of-useeffect-c27ea1e2a5a3)*
