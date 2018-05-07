Minimizing lookup times is a central problem in data management, particularly for large applications. Linear lookup time is bad. Sublinear lookup time is better. Constant lookup is ideal.

Client-side, in the context of a Redux-managed application, the problem becomes a matter of defining the shape of app state.

This article explores two avenues for better organizing such front-end state:
(1) serializing data from the back end, reducing to fully-indexed state shape via utility reducers, and
(2) caching data on the front end, in particular by utilizing [the Reselect library](https://github.com/reduxjs/reselect) in React-Redux `mapStateToProps` functions to create or look up "memos" of chunks of derivable data.

## How to serialize front-end state
Let's say your app state includes data about users, posts, and comments. Maybe you've stored this all in one ginormous `blogPosts` array (see [the Redux docs example](https://redux.js.org/recipes/structuring-reducers/normalizing-state-shape)). Or maybe you've split the data up, but are still storing `users`, `posts`, and `comments` as arrays of users, posts, and comments. At best, your state might look like this:
```js
const initialState = {
  users: [ ],
  posts: [ ],
  comments: [ ]
}
```
Under the above arrangement, finding a particular user, post, or comment can be an O(n) operation. Womp womp.

The problem has to do with indices. In an array, indices are arbitrary markers for order. But you don't generally look something up according to an arbitrarily-derived spot. It's arguably impossible to do so directly. (An interesting case is the divvying up of information by page in a book, in which case there also generally exist mechanisms for associating content with pages directly, to which extent you're able to actually find said info.)

Rather than with arbitrary order, at least to the extent to which language is an exercise in making meaning, we humans tend to better associate information with other semantically relevant information. (Sorry, machine readers; this article is not for you.)

At the database level, *the* semantically-relevant marker is "id". The hypothetical blog post app referenced above probably has database tables for users, posts, and comments. Unless you're doing really funky things at the database or ORM level, these tables are all *indexed* by id. The major implication here is potential constant lookup time for anything, so long as you're looking it up according to the field by which it's been indexed.

Treating Redux store state as a kind of "front-end database" you can similarly use indexing to your advantage on the client, using hashes or key-value mappings for data. A "key" here represents information semantically relevant to an associated set of values. Using a entity's id as a key, individual entity lookups become a constant time operation. (Feel free to use a different parameter as a key, but note you'll want something unchanging and unique.)

To achieve O(1) lookup time by id, we could change the shape of our `initialState` above in the following way:
```js
const initialState = {
  users: {
    id1: { ... },
    id2: { ... },
    //etc.
  },
  // same for posts
  // same for comments
}
```

As Nick Sweeting notes in ["Using the Redux Store Like a Database,"](https://hackernoon.com/shape-your-redux-store-like-your-database-98faa4754fd5) it can prove useful to include an entity's id in the value object for that entity, as well as as the key itself. I.e.,
```js
  users: {
    id1: { id: id1, ... },
    id2: { id: id2, ... },
    //etc.
  },
```

Perhaps the above is enough. We *could* do one better. What if we want to map over all users? It's not exactly self-documenting to write "Object.values(users.byId).map(user => ... )." Or what if we want to reference all users by their ids? Again, not self-documenting to write "Object.keys(users.byId)." There's a level of inference that has to be made about what the keys and values of the users.byId object are supposed to represent, data logic we don't necessarily want to conflate with logic about our views.

Instead, we can add to our state in the following way:
```js
const initialState = {
  users: {
    byId: {
      1: { ... },
      2: { ... },
      //etc.
    },
    allIds: [ 1, 2, ... ]
  }
  // same for posts
  // same for comments
}
```
We have now "serialized" each slice of our front-end state in an array holding all ids tied to values for that slice of state. (Roughly, serialization in this context refers to the reduction of a particular entity to that entity's index: user => userId. Deserialization refers to the opposite progression: userId => user.)

Subsequently, you can map over users as follows:
```js
users.allIds.map(id => {
  const currentUser = users.byId[id];
  ...
});
```

Finally, you can make your life easier on the front end by putting state together as above from the back end through the use of utility reducers for routes.

Here's how we would re-organize an array of objects--i.e., the data shape you'd get back from the database when querying for all of anything--into a fully-indexed slice of state:
```js
const indexState = (dataArray, key = 'id') => {
  const keyTitle = key[0].toUpperCase() + key.slice(1);
  const dataByKey = 'by' + keyTitle;
  const allKeys = 'all' + keyTitle + 's';

  return dataArray.reduce((indexedState, nextItem) => {
    indexedState[dataByKey][nextItem[key]] = nextItem;
    indexedState[allKeys].push(nextItem[key]);
    return indexedState;
  }, {
    [dataByKey]: {},
    [allKeys]: []
  });
}
```
The above function takes two arguments. `dataArray` is the array of data you get from the database. `key`, an optional argument, refers to the field of this slice of state that you want to index. It defaults to `id`.

The first three lines inside the `indexState` function just format the key name properly for the output object. This means that if, for instance, you wanted to index on a "name" instead of an "id" field, you'd get back an object with "byName" and "allNames" fields (as opposed to "byId" and "allIds").

Subsequently, `indexState` reduces data array input to an object with the shape you want. Within this output object, there is a nested object literal containing entries with keys that are the kind of keys you want to use, and a nested array containing just the keys themselves.

You might add this extra piece of logic to your routes as follows (example below uses an Express/Sequelize stack):
```js
router.get('/', (req, res, next) => {
  User.findAll()
    .then(users => res.status(200).json(indexState(users)))
    .catch(next);
});
```

If you're interested further in the topic of indexing Redux store state and/or are confused by anything above, I'd certainly recommend Sweeting's article. If you're thinking, "Why would I ever need to optimize in this way?" you may have a point--indexing can be overkill. And if, like me, you like reveling in the abstracted possibility of having to implement indexing, read on! This article may be for you.

## How to memoize derived data
Great: you've indexed your front-end state. But what if, in addition to accessing a value or values off state, you want to actually do something with that information before presenting output to a user?

Consider a playlist. Let's say a half-dozen songs from such a playlist can be saved or "cached" on the client at any one time. The idea is to be able to play any one of those half-dozen songs for a user as soon as possible after that user requests one of those songs, while minimizing database queries. It follows that the act of playing a saved song becomes a matter of querying its cached location in client memory.

How did to go from playlist to half-dozen songs? In a React-Redux context, playlist info is presumably on state. You'd then filter the original playlist data though a "selector" function that might include making background calls to an audio database to fetch the six cachable songs themselves.

In the context of React-Redux, `connect`'s selector function is conventionally `mapStateToProps`. As Adam Rackis, quoting Redux docs, notes in ["Querying a Redux Store"](https://medium.com/@adamrackis/querying-a-redux-store-37db8c7f3b0f), "This function [`mapStateToProps`] takes the global Redux store’s state, and returns the props you need for the component. In the simplest case, you can just return the state given to you (i.e., pass identity function), but you may also wish to transform it first."

Ideally, you could save or cache the results of such a "selector" function. This "I/O" brand of caching is known as *memoization*. Note this process only works if the output data is derivable from the input data!

The Reselect library allows for such memoized caching of derived data. We'll see below how it does so. If the following leaves you confused, I highly encourage you to check out Dan Parker's ["React, Reselect and Redux"](https://medium.com/@parkerdan/react-reselect-and-redux-b34017f8194c) which basically picks up here and quite possibly offers a much better explanation of the following information than I am currently capable of providing.

That said, let's roll! The following overviews the three levels of selectors feeding into the selector pattern of which Reselect makes use:
(1) The actual filtering selector you wish to apply to derive data;
(2) `createSelector`, Reselect's memoizing selector for (1); and
(3) React-Redux's `mapStateToProps` selector.

### (1) This is a selector function
A selector function is basically a filter:
```js
const selector = something => something.more.specific;
```
### (2) This is how Reselect wraps around a selector function
Reselect is a selector-*wrapper*:
```js
import { createSelector } from 'reselect';

const getSelectedState = createSelector(
  [ selector ],  // here we are passing in `selector` itself, rather than a variable named "selector"
  selector => selector
);
```
Yes, all `selector`s refer to the same selector function. The selector function (a) specified as the first argument to Reselect's `createSelector` wrapper is then (b) passed into the second argument's function as the argument for that function. That callback function then returns (c) that same selector function itself.
### (3) This is how Reselect's selector-wrapper gets used in a React component
Subsequently, you can use `getSelectedState` within the React-Redux-connective `mapStateToProps` selector:
```js
const mapStateToProps = state => {
  return {
    selectedState: getSelectedState(state)
  }
}
```
What's going on?

First, let's see what we're calling:
`mapStateToProps` selector --> `getSelectedState` selector --> `selector` selector

Now, let's review what we're feeding into each function:
`state` --> `selector` --> "`something`"

And this is what we get out:
`something.more.specific`; i.e., a slice of `something`

So the overall pattern is:
(1) Input = `state`
(2) ...
(3) Output = `something.more.specific`

Memoizing (3) thus allows for skipping over (2) so long as (1) remains the same.

The trick is that that "`something`" *also* refers to state. "`state`" and "`something`" *can* be one and the same. So, the above is equivalent to the following pattern:

##### If `state` !== "`something`",
##### Then `state` --> `createSelector([ selector ], selector => selector)` --> `state` --> `state.more.specific`
##### Else, if `state` === "`something`",
##### *Then* `state` --> `something.more.specific` === `state.more.specific`

Under the hood, as Parker notes, "Reselect handles the memoization." This makes for very convenient React-Redux `connect`ed behavior: "Whenever an action is called anywhere in the application, all mounted and connected components call their `mapStateToProps` function[s]." Reselect, in turn, "will just return the memoized result[s] if nothing has changed."

Again I encourage you to check out Parker's article as it goes into specifics of more advanced behavior, namely the additional (fourth level of) "makeGetSelectedState" and "makeMapStateToProps" selectors, which function as wrappers around `getSelectedState` and `mapStateToProps`, respectively, and ought to be used to be able to reuse memoized slices of state across different components.

## How to consider your options going forward
Is all of the above overkill? It could be. If you're not planning to "scale," the answer's quite probably yes. That said, while not wanting to prematurely optimize, utilizing the above patterns can set the groundwork for effective scaling of an application. And they certainly seem to be useful for larger applications (Reselect tends to get cited in such a capacity).

All *that* said, I still would like at some point in the future to benchmark lookup performances given various sizes of input. I'm genuinely not sure at this point where the "cutoff" is, or even if there can be a hard cutoff given other constraints such as variable wifi connection strength. I'll leave that for another article.

Still, my main takeaway is the abstracted possibility of having to implement indexing and memoization for derived data in a relatively large application. There will hopefully come points at which these are the kinds of optimizations I'll need to implement, and this exploration leaves me elated to relish such opportunities.
