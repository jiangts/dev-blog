---
layout: post
title: Isolating Mithril components
categories: [mithril, api-design, snippet]
typora-root-url: ../images
---



Recently I've been building a new site, [EasyVirtualChoir](https://easyvirtualchoir.com/) that helps folks sing together remotely. It's been getting a fair amount of usage during the pandemic.

When starting the site, I decided to try out [Mithril](http://mithril.js.org/), and I can say that I've loved it so far! I might write a separate post on Mithril, but my tldr summary of why I love it is: it's simple, unopinionated, and **it doesn't get in the way**. I think after years of trying to write idiomatic React code, every time I wanted to try a code snippet online (say, of some new HTML5 API) I'd be taken down a journey of trying to mash functional and procedural/OO code styles together. It was tiring, and led to me leaving lots of unidiomatic code in my codebase...

---

Anyways, as I was making EVC, I found myself creating a lot of progress loaders for long download tasks or long server tasks.

To make a progress loader "move" in Mithril, you need to call `m.redraw()`... but doing this all the time can cause other problems -- since redraws may prevent local updates to the DOM (e.g. typing into an `<input>` field), or otherwise cause the DOM to redraw when it's undesirable.

### "Self-sufficient" components

To fix this, I googled something like "isolate mithril render" and found this GH issue "[Subtree rendering proposal](https://github.com/MithrilJS/mithril.js/issues/1907)". While following the convo, I found Isiah's [repo](https://github.com/isiahmeadows/mithril-helpers/blob/master/docs/self-sufficient.md) with support for "self-sufficient components", that receive their own redraw function. I took it, and decided to use it like this:

```javascript
const ExampleLoader = () => {
  let redraw
  let value, max
  socket.on('progress', (data) => {
    value = data.value
    max = data.max
    if(redraw) { redraw() }
  })
  return {
    view: (vnode) => {
      return m(m.helpers.SelfSufficient, {
        root: m("div"),
        view: state => {
          redraw = state.redraw
          return [
            // we are passed in a state.redraw() function (which redraws only the SelfSufficient component)
            // and a couple others functions, documented here:
            // https://github.com/isiahmeadows/mithril-helpers/blob/master/docs/self-sufficient.md
            max && m("progress", { value, max })
          ]
        }
      })
    }
  }
}
```

As you can see, in the way I use mithril I realllly capitalize on function closures :smile:

<br>

This was pretty cool, and I successfully used it for a component or two.

But, it was still unsatisfying: I had to remember the new API of `SelfSufficient`, and it's convoluted at best. The new redraw function only exists on `state`, but that's only in scope of the view function itself. In all the use cases I had, I needed redraw outside of the view function, hence the weird stuff with `if(redraw) { redraw() }`.

<br>

### "Isolated components"

So, I did some home cooking. I knew the API I eventually wanted would be a "decorator" pattern: instead of re-writing a component, I want to use some simple function wrapper on the outside to convert it into a component whose `render` is isolated.

One might think doing this would be pretty hard. I thought it would be, too... But after some tinkering, here's what I came up with:

#### Usage
```javascript
// a normal component, except for: 
// getting redraw from the component's fn argument, 
// and using that function instead of m.redraw
const ExampleLoader = ({ redraw=m.redraw }) => {
  let value, max
  return {
    oninit: vnode => {
      socket.on('progress', (data) => {
        value = data.value
        max = data.max
        redraw()
      })
    },
    onremove: vnode => {
      socket.off('progress')
    },
    view: (vnode) => {
      return max && m("progress", { value, max })
    }
  }
}

const IsolatedLoader = m.isolate(ExampleLoader, '.loader-container')
// Now you may either use m(ExampleLoader) which employs global redraws
// or m(IsolatedLoader) where its redraws are local.
```

#### Implementation
todo: put code up with comments

#### Note on self-sufficient components
todo: note bugs and fixes in the public implementation
