{:title "Debugging with MutationObserver"
 :layout :post
 :tags ["javascript" "debugging"]}

## Turns out JavaScript's MutationObserver is pretty useful for debugging

I recently was trying to figure out why a specific element wasn't showing up on the page. The code was from a legacy system making liberal use iframes and a very interesting [jQuery plugin](https://github.com/codrops/BookBlock) that brings the authentic experience of flipping through a paper book to the screen...sort of!

I won't pretend to know too much about it, but this plugin, BookBlock, is supposed to `show` the first page once the `bookblock` instance is created. It wasn't doing that. **_Why?!_**

This bug seemed intermittent until we figured out it could only be reproduced by loading the page such that the iframe containing the BookBlock markup was initially off-canvas. If we loaded the page with the iframe in full view, the BookBlock code would set the `display` property of the first page of the "book" to `block`; otherwise, it would remain `none`.

Was something interfering with `display` once the BookBlock code had already run? Or was it just never running? Could I find out without having to read and debug unfamiliar jQuery core/plugin code?

## Enter MutationObserver

`MutationObserver` is a JavaScript constructor that "provides developers a way to react to changes in a DOM," according to [MDN](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver). You can get real specific, and tell a specific observer instance to watch for changes on a single element. You can even tell it to watch specific attributes. Eureka!

I put the following code in the markup, right after the BookBlock target container:

```
</div><!-- end #bookblock-target -->
<script>
var observer = new MutationObserver(function(records) {
    console.log('mutation observed!', records[0]);
});
var page1 = document.getElementById('page1');
observer.observe(page1, {attributes: true, attributeFilter: ['style']});
</script>
```

This basically says: *Hey, JS, watch the `#page1` element for me!* The object literal (`{attributes: true, attributeFilter: ['style']}`) that gets passed into the `observe` method is me telling it: *Oh, and just watch for changes to **attributes** on that element, and I really only care about the `style` attribute. K thx!*

Then I ran my two test cases again. When loading the page with the iframe directly visible, the "mutation observed!" showed up in the console right away, along with the `MutationRecord` object at `records[0]` (an array of these gets passed in to give your observers more context about what happened, which is nice). Then, when loading with the iframe off-canvas, no console message shows up. Bingo!

So we know that the call to `$.show()` in the bookblock code just isn't running. Cool! Why not?

It turns out that the iframe's (lack of) visibility caused some parameter to the `$.bookblock()` method to not be set properly, preventing `$.show()` from being called. Granted, we could have figured this out by un-minifying the BookBlock code and debugging it directly. But `MutationObserver` let us a quickly confirm that it was something *in or upstream of* the lookbook code, rather than something downstream resetting it. This in turn saved us from spending time trying to understand BookBlock implementation details just to figure out where to start debugging.



