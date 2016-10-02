{:title "Creating Generalized Column Templates in Twig"
 :layout :post
 :tags ["twig" "templates"]}

## Evenly render a dynamic list across columns, in pure Twig

Columns are such an effective design solution for so many kinds of content that you really can't do front-end web development without them. Lists are one obvious use-case. To make the most of valuable real estate on a desktop screen, you'll probably want to break long lists up into two or more columns:

```
<div class="row">    
    <div class="col">
        <ul class="menu">
            <li>List Item 1</li>
            <li>List Item 2</li>
            <li>List Item 3</li>
        </ul>
    </div>
    <div class="col">
        <ul class="menu">
            <li>List Item 4</li>
            <li>List Item 5</li>
            <li>List Item 6</li>
        </ul>
    </div>
</div>
```

This lets us easily style each half of the list as its own column. So far so good. But what if we're dealing with dynamic data? Splitting the list into chunks of three is easy, but what if we don't know ahead of time how long our list will be?

## Let's do maths

The short answer is simple division: code our templates to figure out on the fly how many items to put in each list. You can do this quite easily in Twig using the `batch` filter (which uses PHP's [array_chunk()](http://php.net/manual/en/function.array-chunk.php) function behind the scenes):

```twig
{# Get chunk length by rounding items per column up to nearest whole #}
{% set `chunk_len` = ( (items|length)/2 | round(0, 'ceil') ) %}
 
{# Divide the items into chunks #}
{% set chunks = ( items | batch(`chunk_len`) ) %}
 
{% for chunk in chunks %}
    <div class="col">
        <ul class="menu">
            {% for item in chunk %}
                <li>{{ item.text }}</li>
            {% endfor %}
        </ul>
    </div>
{% endfor %}
```

Do you see what's happening here? We first pipe our collection of items through the `length` filter to count them, then divide that number by two, rounding up to the nearest whole. Given the way `array_chunk()` works, we round up so that, if we happen to split an odd number of items into chunks, we display the longer chunk on the left.

Now that we have our `chunk_len`, we simply pipe our list into the `batch` filter. This says: "split my items into chunks of length `chunk_len` at most." After that, it's a simple matter of nested loops to output the markup for each column.

## Abstraction

This is useful, but we can do better. We don't need to repeat the calculation logic in every place we use this pattern. We can abstract the calculation piece of the puzzle away from the piece that renders each chunk/column. How? By telling Twig three things:

* This is my list of items
* This is how many chunks I want
* Here's the template to use to render each chunk

So let's first setup a partial that does the calculation for us. Save this to a shared template, say `partials/chunks.twig`:

```twig
{# Get chunk length by rounding items per column up to nearest whole #}
{% set `chunk_len` = ( (items|length)/2 | round(0, 'ceil') ) %}
 
{# Divide the items into chunks #}
{% set chunks = ( items | batch(`chunk_len`) ) %}
 
{% for chunk in chunks %}
    {% include single_template with { chunk: chunk } only %}
{% endfor %}
```

Notice that this is basically the same code as before, except that this template is no longer responsible for rendering each chunk itself. In fact, it's effectively de-coupled from any rendering at all! This is great because it means we can use it to render any list of dynamic length, using any markup, and our markup doesn't have to worry about the calculation side, either.

Next, we'll set up a partial for actually rendering the `chunk`, or column. Let's save this one to `partials/column.twig`:

```twig
<div class="col">
    <ul class="menu">
        {% for item in chunk %}
            <li>{{ item.text }}</li>
        {% endfor %}
    </ul>
</div>
```

Finally, to tie it all together, we'll simply call our `chunks` partial and pass it the *name* of our single-chunk partial, in this case the `partials/column.twig` template we just set up:

```twig
{# assume our PHP set up list_items as ['List Item 1', ... , 'List Item 6'] #}
{% include "partials/chunks.twig" with {
    items: list_items,
    num_chunks: 2,
    single_template: "partials/column.twig"
} %}
```

As before, this should output:

```
<div class="row">    
    <div class="col">
        <ul class="menu">
            <li>List Item 1</li>
            <li>List Item 2</li>
            <li>List Item 3</li>
        </ul>
    </div>
    <div class="col">
        <ul class="menu">
            <li>List Item 4</li>
            <li>List Item 5</li>
            <li>List Item 6</li>
        </ul>
    </div>
</div>
```

Now, if we have a slightly different use-case with a different number of columns, we can simply change the `num_chunks` variable we pass to `chunks.twig`. If we want to render our items in nested `div`s instead of lists, we can just set up a new partial to handle that. We never have to tweak our `chunks.twig` or `column.twig` partials at all!

This approach is powerful enough that we're really not even talking about columns anymore. It's really any number of *elements* among which you want to quasi-evenly divide some number of *items.* Abstraction FTW!

