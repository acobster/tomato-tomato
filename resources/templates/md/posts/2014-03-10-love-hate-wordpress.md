{:title "My Love/Hate Relationship with WordPress"
 :layout :post}

## WordPress is a great first web technology to learn. Wait, no it's not.

WordPress is amazing. When I started ~~cobbling websites together~~ my professional web development career, I thought it was the greatest thing since, like, [PHP](https://eev.ee/blog/2012/04/09/php-a-fractal-of-bad-design/).

*WordPress lets you do anything!* I would think, bright-eyed.

*Yeah...* I think now. *That's kinda the problem.*

Now that I've had some time to reflect and, more importantly, actually write and maintain WordPress code, I think it's obvious...

## Why WordPress Sucks

Take the data "layer" for instance. It's not really a layer (and to be fair, [it doesn't pretend to be](https://codex.wordpress.org/Class_Reference/wpdb)). It's a global object. Here's how you'd perform a custom query against your WP database:

```php
global $wpdb;
$wpdb->get_results(
    $wpdb->prepare(
        'SELECT * FROM wp_posts WHERE ID = %d',
        123
    )
);
```

You probably wouldn't run this query directly through the `$wpdb` object in practice, since WordPress provides abstractions for simple data access like this. But the point stands that "Oh, I know, I'll just make it a global object" is not really a design decision so much as a design non-decision. Or, if you want to be harsh about it, it's a decision *not to have* a design. "Rather than code up a whole OO architecture, I'm just going to leave this global state lying around and let people do whatever they want with it."

The problem with this, in case you don't know, is that code that relies on global state is really brittle, meaning you can break things in other areas that 

At least this ~~problem~~ design is [well documented](https://codex.wordpress.org/Global_Variables).

Let's talk about their core API: the way WordPress lets you extend its functionality. Say I wanted to, oh I dunno, [modify some content](https://codex.wordpress.org/Plugin_API/Filter_Reference/the_content) before it gets rendered. That seems like a pretty basic use-case, right? Here's what I might do:

```php
add_filter( 'the_content', 'my_the_content_filter', 20 );
/**
 * Add a icon to the beginning of every post page.
 *
 * @uses is_single()
 */
function my_the_content_filter( $content ) {

    if ( is_single() )
        // Add image to the beginning of each page
        $content = sprintf(
            '<img class="post-icon" src="%s/images/post_icon.png" alt="Post icon" title=""/>%s',
            get_bloginfo( 'stylesheet_directory' ),
            $content
        );

    // Returns the content.
    return $content;
}
```

Now, if I'm a regular WP dev, I add that to my `functions.php` because that's where functions and stuff go. If you're not familiar, that code snippet is straight from the "Codex," WordPress's developer doc site.

Later, if you want to, say, modify how WordPress runs a query, you'd do something kinda like this except with a different hook and your function takes a different thing to modify and then you stick that in `functions.php` as well.

Let me just end this section by pointing out that the WordPress motto is "Code is Poetry." Ugh.

## Why WordPress Is Amazing

### ALL THE PLUGINS

There are [so many plugins](https://wordpress.org/plugins/).

### ALL THE DOCS

The [Codex](https://codex.wordpress.org/Plugin_API/Filter_Reference/) is an enormous wealth of information. It's obvious that the WordPress core devs and supporting volunteers really, really care about educating people 

### ALL THE HOOKS

### ALSO

* It's stable and fairly secure, I think
* 

## Why WordPress Is a Great First Web Framework...

* you can throw a decent WP site together fast
* It lets you write jQuery and CSS and not worry about the gory details of how it gets there
* you can do this, except

## ...And Why It's Not

* Docs and ecosystem teach you anti-patterns
* ...so does the general structure
* ......so does all the code

## ...And Yet...

I'm glad I had the experience I did with WordPress, even if I felt like pulling my hair out some of the time. It helped me keep a roof over my head, and it gave me my first taste of learning the ins and outs of a huge system. 

But do I feel like it gave me a solid foundation in web development? best practices? software architecture? No way. A WordPress site, and the supporting core software, is an excercise in how *not* to build something.

So if you're a newbie web dev trying to carve out your place in the professional world and a prospective client wants a WordPress site, by all means, make them a WordPress site. But expect trouble. And if you ever want to graduate to a saner system, expect to relearn everything.
