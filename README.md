# backbone-architecture-101

A short, opinionated guide to Backbone architecture.

### The Basic Interaction

![](https://cldup.com/Nk908IzFJx.png)

The "Basic Interaction" is a simple pattern that occurs many times throughout a Backbone application. In fact, the
majority of your application will likely be the Basic Interaction. The basic interaction describes how two objects
can directly communicate with one another. These two objects will be referred to as the "Parent" and the "Child."

The parent and child objects have a single defining quality each: the parent creates the child, and the child
is created by the parent.

An example of this are two views. Imagine a view that is a tab navigation bar. When a user clicks a tab, the view
creates a child view that displays the tab contents. In this example, the tab navigation bar is the parent,
and the tab contents view is the child.

**Principles of the basic interaction**

- The parent creates the child
- The child can itself also be a parent, by creating its own children

- The parent can directly access "public" methods on the child
- The child should not directly access any methods on the parent

- The parent should listen to, and react to, events emitted by the child
- The child should not listen to, or react to, events emitted by the parent

Here's an example of the basic interaction, using a custom API for rendering children and listening
to child events (note that nither of these are included in Backbone out-of-the-box).

```js
// The `TabNavView` is the parent of our `TabContentsView`.
var TabNavView = BaseView.extend({

  // When the tab is clicked, the contents view is created
  onClickTabContents() {
    this.showChildView('.tab-contents', new TabContentsView());
  },

  // Listen to when the child view is destroyed, and show a default
  // "SelectTabsView"
  childEvents: {
    'destroy': 'showSelectView'
  },

  // Shows a default view that might tell the user about the different tabs
  showSelectView() {
    this.showChildView('.tab-contents', new SelectTabsView());
  }
});
```

**Deeply nested trees**

Sometimes a child needs to communicate an event up several levels of views. "Event forwarding" is one way to do this.
Event forwarding is when an object listens to another object's event, then forwards it along for the sole purpose of
another object getting that event. Event forwarding requires a lot of boilerplate code, which I try to avoid. Instead,
I use Backbone.Radio to "jump" up and down view trees.

Here's an example of what event forwarding might look like:

```js
// If your event handler just re-emits the event, then you're doing event forwarding.
// I do not like event forwarding.
this.listenTo(myChild, 'some:event', () => {
  this.trigger('some:event');
});
```

Here's an example of using Backbone Radio instead:

```js
// deeply-nested-child.js
var DeeplyNestedChild = View.extend({
  someMethod() {
    // Use a channel with a relevant name; probably related to the feature that the views are for
    var viewChannel = Radio.channel('view-tree');
    // Emit the event on the channel
    viewChannel.trigger('some:event');
  }
});

// great-grandparent.js
var GreatGrandparent = View.extend({
  initialize() {
    // Access the channel from this view, too
    var viewChannel = Radio.channel('view-tree');
    // Listen and respond to events on the channel
    this.listenTo(viewChannel, 'some:event', this.onSomeEvent);
  }
});
```

### The "root" parent

In the above section, I outlined a few tips on how two objects typically interact in Backbone. The foundation of the basic
interaction described above is that parents create children. But where does the first parent come from?

The solution is the router. Or, more specifically, a "route."

All client side applications follow the same underlying algorithm. Simply, it looks like this:

1. The user navigates to a URL
2. The Router matches the URL to a Route
3. The Route optionally fetches data
4. The Route shows a view

Each of these sections can be broken down further:

#### The user navigates

This is one of two things:

1. The user loads the application at a given URL
2. The user clicks a link

In the first situation, the router recognizes the URL, and sends it off to the corresponding Route. In the second case, I strongly
recommend that you build a system that automatically hooks your links up to the Router. Ember's `link-to` helper is the best
system I know of that does this, and similar constructs can be found in Angular and React.
[Backbone.Intercept](https://github.com/jmeas/backbone.intercept) is the current solution I use, as it was quicker to write than a link-to
abstraction.

Although the 'inputs' in either situation are distinct, the "output" is the same: a URL is passed off to the Router.

#### The Router matches the URL to a Route

Backbone is the last remaining popular client side application to not used nested routes. Because of this, and because of the messy way
that Backbone's Router works, 

#### The Route optionally fetches data

At this point, the route was matched. The first thing that the route does is fetch data, if it needs to. This should be done through a data
layer. The goal of the data layer is to manage caching. The route is ignorant as to whether the data is cached or not: it simply requests it
and resolves a Promise when it gets it back.

#### The Route shows a view

We now have all of the data that we need for the route, so the last thing to do is to show a view. This is the first parent that begins our
basic interaction. This view can go on to make children views and wire up interactions.