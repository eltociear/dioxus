# Signals: Skipping the Diff

In most cases, the traditional VirtualDOM diffing pattern is plenty fast. Dioxus will compare trees of VNodes, find the differences, and then update the Renderer's DOM with the updates. However, this can generate a lot of overhead for certain types of components. In apps where reducing visual latency is a top priority, you can opt into the `Signals` api to entirely disable diffing of hot-path components. Dioxus will then automatically construct a state machine for your component, making updates nearly instant.

Signals build on the same infrastructure that powers asynchronous rendering where in-tree values can be updated outside of the render phase. In async rendering, a future is used as the signal source. With the raw signal API, any value can be used as a signal source.

By default, Dioxus will only try to diff subtrees of components with dynamic content, automatically skipping diffing static content.

## What does this look like?

Your component today might look something like this:

```rust
fn Comp(cx: Context<()>) -> VNode {
    let (title, set_title) = use_state(&cx, || "Title".to_string());
    cx.render(rsx!{
        input {
            value: title,
            onchange: move |new| set_title(new.value())
        }
    })
}
```

This component is fairly straightforward - the input updates its own value on every change. However, every call to set_title will re-render the component. If we add a large list, then every time we update the title input, Dioxus will need to diff the entire list, over, and over, and over. This is **a lot** of wasted clock-cycles!

```rust
fn Comp(cx: Context<()>) -> VNode {
    let (title, set_title) = use_state(&cx, || "Title".to_string());
    cx.render(rsx!{
        div {
            input {
                value: title,
                onchange: move |new| set_title(new.value())
            }
            ul {
                {0..1000.map(|f| rsx!{
                    li { "{f}" }
                })}
            }
        }
    })
}
```

Many experienced React developers will just say "this is bad design" - but we consider it to be a pit of failure, rather than a pit of success! That's why signals exist - to push you in a more performant (and ergonomic) direction. Signals let us directly bind values to their final place in the VirtualDOM. Whenever the signal value is updated, Dioxus will only the DOM nodes where that signal is used. Signals are built into Dioxus, so we can directly bind attributes of elements to their updates.

We can use signals generated a two-way binding between data and the input box. Our text input is now just a two-line component!

```rust
fn Comp(cx: Context<()>) -> VNode {
    let mut title = use_signal(&cx, || String::from("Title"));
    cx.render(rsx!(input { value: title }))
}
```

For a slightly more interesting example, this component calculates the sum between two numbers, but totally skips the diffing process.

```rust
fn Comp(cx: Context<()>) -> VNode {
    let mut a = use_signal(&cx, || 0);
    let mut b = use_signal(&cx, || 0);
    let mut c = a + b;
    rsx! {
        input { value: a }
        input { value: b }
        p { "a + b = {c}" }
    }
}
```

Do you notice how we can use built-in operations on signals? Under the hood, we actually create a new derived signal that depends on `a` and `b`. Whenever `a` or `b` update, then `c` will update. If we need to create a new derived signal that's more complex that a basic operation (`std::ops`) we can either chain signals together or combine them:

```rust
let mut a = use_signal(&cx, || 0);
let mut b = use_signal(&cx, || 0);

// Chain signals together using the `with` method
let c = a.with(b).map(|(a, b)| *a + *b);
```

## Deref and DerefMut

If we ever need to get the value out of a signal, we can simply `deref` it.

```rust
let mut a = use_signal(&cx, || 0);
let c = *a + *b;
```

Calling `deref` or `derefmut` is actually more complex than it seems. When a value is derefed, you're essentially telling Dioxus that _this_ element _needs_ to be subscribed to the signal. If a signal is derefed in _no_ context, the entire component will be subscribed and the advantage of skipping diffing will be lost. Dioxus will throw an error in the console when this happens to tell you that you're using signals wrong, but your component will continue to work.

## Global Signals

Sometimes you want a signal to propagate across your app, either through far-away siblings or through deeply-nested components. In these cases, we the Dirac: Dioxus' first-class state management toolkit. Dirac atoms implement the Signal API. This component will bind the input element to the `TITLE` atom.

```rust
const TITLE: Atom<String> = || "".to_string();
const Provider: FC<()> = |cx| {
    let title = use_signal(&cx, &TITLE);
    rsx!(in cx, input { value: title })
};
```

If we use the `TITLE` atom in another component, we can cause updates to flow between components without calling render or diffing either component trees:

```rust
const Receiver: FC<()> = |cx| {
    let title = use_signal(&cx, &TITLE);
    log::info!("This will only be called once!");
    rsx!(in cx,
        div {
            h1 { "{title}" }
            div {}
            footer {}
        }
    )
};
```

Dioxus knows that the receiver's `title` signal is used only in the text node, and skips diffing Receiver entirely, knowing to update _just_ the text node.

## Signals and Iterators

Sometimes you want to use a collection of items. With Signals, you can bypass diffing for collections - a very powerful technique to avoid re-rendering on large collections.

Dioxus automatically understands how to use your signals when mixed with iterators through Deref/DerefMut. This lets you efficiently map collections while avoiding the re-rendering of lists. In essence, signals act as a hint to Dioxus on how to avoid un-necessary checks and renders, making your app faster.

```rust
const DICT: AtomFamily<String, String> = |_| {};
const List: FC<()> = |cx| {
    let dict = use_signal(&cx, &DICT);
    cx.render(rsx!(
        ul {
            {dict.iter().map(|(k, v)| rsx!{
                li { "{v}" }
            })}
        }
    ))
};
```

## Remote Signals

Apps that use signals will enjoy a pleasant hybrid of server-side and client-side rendering.

```rust

```

## How does it work?

Signals internally use Dioxus' asynchronous rendering infrastructure to perform updates out of the tree.