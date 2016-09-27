{:title "My Love/Hate Relationship with WordPress"
 :layout :post}

## WordPress is a great first web technology to learn. Wait, no it's not.

WordPress is amazing. When I started ~~cobbling websites together~~ my professional web development career, I thought it was the greatest thing since, like, [PHP](https://eev.ee/blog/2012/04/09/php-a-fractal-of-bad-design/).

*WordPress lets you do anything!* I would think, bright-eyed.

*Yeah...* I think now. *That's kinda the problem.*

Now that I've had some time to reflect and, more importantly, actually write and maintain WordPress code, I think it's obvious...

## Why WordPress Sucks

**Warning: If you can't tell by now, what follows is a scathing rant. I've made a feeble effort to disguise it as beginner advice and even add some nuance! You'll probably see right through that, but hopefully somebody somewhere will find this helpful anyway.**

Take `$wpdb`. It's not really a layer (and to be fair, [it doesn't pretend to be](https://codex.wordpress.org/Class_Reference/wpdb)). It's a global object. Here's how you'd perform a custom query against your WP database:

```php
global $wpdb;
$wpdb->get_results(
    $wpdb->prepare(
        'SELECT * FROM wp_posts WHERE ID = %d',
        123
    )
);
```

You'd probably use an abstraction like `get_post` for simple data access like this in practice, but the point stands: "Oh, I know, I'll just make it a global object" is not really a design decision so much as a design non-decision. Or, if you're feeling less charitable like I am, it's a decision *not to have* a design. "Rather than code up a whole OO architecture, I'm just going to leave this global state lying around and let people do whatever they want with it." Um, thanks?

The problem with this, in case you don't know, is that code that relies on global state is really brittle, meaning you can break it easily, even from other parts of the code that have nothing to do with it. Say, for instance, that you wanted to link to a specific post in one of your templates. You might do something like:

```php
$post = get_post(345); ?>
<a href="<?= get_permalink($post->ID) ?>"><?= $post->post_title ?></a>
```

Oops. If you were in the process of rendering a different post before linking to this one, you just overwrote that. The code that runs subsequently will think you're rendering post `345`, and the content before and after this link will be mismatched. This can manifest itself in subtle ways, for example in links to next/previous posts. These bugs can be very hard to track down.

"Straw man argument!" the WordPress veterans cry. "Any dev worth their salt knows you can just use a different variable name, or use a new `WP_Query` object to get the data, or..."

Yeah. That's my point. In order to be worth your salt, you have to know these things in and out. In other words, you have to know the implementation details of the system you're building on top of just to know how to use its API. That's not just a minor gripe with WordPress; that's the fundamental antithesis to [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming), a core concept of any kind of software design.

At least this ~~problem~~ design is [well documented](https://codex.wordpress.org/Global_Variables).

But this doesn't just have to do with variable names. Lots of things, including examples from the official docs, rely on hooking into the "current" post. `rewind_posts` is a [core WP function](https://codex.wordpress.org/Function_Reference/rewind_posts) for "rewinding" when looping through a collection of posts "in order to re-use the same query in different locations on a page."

Consider an example from the `rewind_posts` doc: rendering all blog posts by a given author. In simple object-oriented land, you'd do something like:

```php
<header>
    <h1><?= $author->name ?></h1>
    <h2><?= $author->description ?></h2>
</header>
<?php foreach ($author->get_posts() as $post) :
    <article>
        <h3><?= $post->title ?></h3>
        <div><?= $post->content ?></div>
    </article>
<?php endforeach; ?>
```

Here we display some info about the author, and then we display their posts. Pretty simple. How does WordPress do it?

```php
   <?php if (have_posts()): ?>
      <header>
       // initialize posts
        <?php the_post(); 
          _e('All Post by :'); echo get_the_author(); ?>
      <?php if (get_the_author_meta( 'description' )): ?>
          <?php the_author_meta('description'); ?>
          <?php endif; ?>
      </header>
       // rewind posts
    <?php rewind_posts(); ?> 
    <?php while (have_posts()): the_post();
        // presumably we display some stuff here
        endwhile; endif;?>
```

Here we do the same thing, but it's not nearly as obvious. What is `the_post`? Turns out it's responsible for setting our global `$post` variable. Why do we need to do that? Turns out `get_the_author` and friends are designed to work under the assumption that you've already set `$post`. So aside from having a terrible name, `the_post` obfuscates what's going on: even as a WP coder of 5+ years, I scratched my head over this example before I realized what it was doing. We iterate through the posts just to get some information that doesn't have to do with the posts directly; then we wind *back* the iteration to actually display each post's info. If we forget that step, we skip displaying the first post.

This gets a lot more complicated when you want to abstract this pattern across reusabled template components (which WordPress recommends you do, to its credit). You might not need to call `rewind_posts` initially, but then one of your 

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

Have you heard of separation of concerns? That's another pretty fundamental principle software that goes hand-in-hand with encapsulation, and forms the basis of the Unix philosophy: do one thing well.

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
