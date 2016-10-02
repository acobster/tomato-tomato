{:title "My Love/Hate Relationship with WordPress"
 :layout :post}

## WordPress is a great first web technology to learn. Wait, no it's not.

WordPress is amazing. When I started ~~cobbling websites together~~ my professional web development career, I thought it was the greatest thing since, like, [PHP](https://eev.ee/blog/2012/04/09/php-a-fractal-of-bad-design/).

*WordPress lets you do anything!* I would think, bright-eyed.

*Yeah...* I think now. *That's kinda the problem.*

Now that I've had some time to reflect and, more importantly, actually write and maintain WordPress code, I think it's obvious...

## WordPress Sucks for Developers

I don't mean that WordPress has no redeeming features or that it isn't good for anything. WordPress is *amazing* for tech-savvy-ish folks who need a simple website or blog with, I don't know, some map features tacked on. Or an event calendar. Or one or two other simple features that just need to *work* and not integrate super tightly.

But once you get into custom functionality land, you need to dive into coding against the WordPress API. Custom WordPress sites are how many people find their way into the world of web development [citation needed], but WordPress is *not* a good way to learn how to code.

**It's not so much that the learning curve is steep, but that the learning curve is so long.** The more complex your requirements, the more coupled to the core's arbitrary implementation details your thinking must become. *OK,* you get to thinking as a WordPress dev, *in order to do X, Y, and Z, I just have to figure out a way to express that in terms of [The Loop](https://codex.wordpress.org/The_Loop).* Put another way, **the WordPress core is too exposed.**

I wouldn't take such issue with this if it weren't for one fundamental deal-breaker: **the core's design is full of anti-patterns.**

### WordPress Teaches Anti-Patterns from the Start

Take `$wpdb`. It's not really a data layer (and to be fair, [it doesn't pretend to be](https://codex.wordpress.org/Class_Reference/wpdb)). It's a global object. Here's how you'd perform a custom query against your WP database:

```php
global $wpdb;
$wpdb->get_results(
    $wpdb->prepare(
        'SELECT * FROM wp_posts WHERE ID = %d',
        123
    )
);
```

You'd probably use an abstraction like `get_post` for simple data access like this in practice, but the point stands: "Oh, I know, I'll just make it a global object" is not really a design decision so much as a design non-decision, a decision *not to have* a design. "Rather than code up a whole OO architecture, I'm just going to leave this global state lying around and let people do whatever they want with it." Um, thanks?

The problem with this, in case you don't know, is that code that relies on global state is really brittle, meaning you can break it easily, even from other parts of the code that have nothing to do with it. Say, for instance, that you wanted to link to a specific post in one of your templates. You might do something like:

```php
$post = get_post(345); ?>
<a href="<?= get_permalink($post->ID) ?>"><?= $post->post_title ?></a>
```

Oops. If you were in the process of rendering a different post before linking to this one, you just overwrote that. The code that runs subsequently will think you're rendering post `345`, and the content before and after this link will be mismatched. This can manifest itself in subtle ways, for example in links to next/previous posts. These bugs can be very hard to track down.

"Straw man argument!" the WordPress veterans cry. "Any dev worth their salt knows you can just use a different variable name, or use a new `WP_Query` object to get the data, or..."

Exactly. In order to be worth your salt, you have to know these things in and out. In other words, you have to know the implementation details of the system you're building on top of just to know how to use its API. That's not just a minor gripe with WordPress; that's the fundamental antithesis to [encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming), a core concept of any kind of software design.

At least this ~~problem~~ design is [well documented](https://codex.wordpress.org/Global_Variables).

### Just Press Rewind

This doesn't just have to do with variable names. Lots of things, including examples from the official docs, rely on hooking into the "current" post. `rewind_posts` is a [core WP function](https://codex.wordpress.org/Function_Reference/rewind_posts) for "rewinding" when looping through a collection of posts "in order to re-use the same query in different locations on a page."

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

Why does this feel so kludgy? Because of The Loop, the central design pattern of WordPress! Notice that in our simple Object-Oriented example above, displaying all the posts by a given author is centered, unsurprisingly, around the `$author` object. We only use a loop when we need to, you know, loop through the posts. Not so in WordPress land: you have to shoehorn the author-specific stuff into The Loop's opinionated approach.

This isn't just a problem with `rewind_posts`. It's a problem with The Loop itself. **Not everything is a loop, but in WordPress, everything has to be.**

### functions.php: The Dumping Ground

There's a file required for every WordPress theme called functions.php. The name says a lot: this is where the WordPress documentation encourages you to dump pretty much any custom behavior that's not unique to one template. Custom data-fetching code gets pasted in right next to custom helper functions for rendering stuff, right alongside stuff that does both within the same function!

Now, sometimes this is *fine.* You probably don't need to have strict Model-View-Controller style separation of concerns for your buddy's hobby website.

But if what you're trying to get into Serious Web Development™, you need a handle on best practices. Among other things, you need to learn:

* design patterns, and when (not) to apply them
* separation of concerns
* dependency management techniques and tools
* how to clearly organize your source code

WordPress teaches you none of this stuff. To its credit, the WordPress core doesn't really have an opinion about these things. That *is* a good thing, I think, as you generally want your core to be as unopinionated as possible so it can stay out of your way. What's not so good is that the most common convention I've seen throughout the WordPress ecosystem (plugins, themes, code snippets, etc.) is *lack of convention.* The most common software design/architecture the WordPress community puts forth as The Way to Do WordPress is *lack of design/architecture.* "Toss this code in your functions.php file" is an all-too-common suggestion for solving any given problem.

## WordPress Is Still Amazing

There are [so many plugins](https://wordpress.org/plugins/)! There are so many [actions](https://codex.wordpress.org/Plugin_API/Action_Reference) and [filters](https://codex.wordpress.org/Plugin_API/Filter_Reference)! You can do anything with them! Again, this is kinda bad, but it's also good. Despite its kludginess, it's very flexible.

The [Codex](https://codex.wordpress.org) is an enormous wealth of information. It's obvious that the WordPress community really, really care about educating people on how to use WordPress.

The WordPress core is fairly stable and secure, as long as you make updates regularly.

If you're a minimally skilled developer, you can throw a decent WordPress site together quickly. It lets you write jQuery and CSS and not worry about the gory details of how it gets there.

But the documentation itself teaches you anti-patterns. So does the ecosystem. So does the general structure and design of the code. An expert web developer can pick up WordPress and hit the ground running after a bit of acclimation. The inverse is not necessarily true: a WordPress expert is *not* an expert web developer. The unique skills that WordPress teaches you are just not that transferrable.

I'm glad I had the experience I did with WordPress, even if I felt like pulling my hair out some of the time. It helped me keep a roof over my head, and it gave me my first taste of learning the ins and outs of a huge system. 

But do I feel like it gave me a solid foundation in web development? best practices? software architecture? No way. A WordPress site, and the supporting core software, is an excercise in how *not* to build something.

So if you're a newbie web dev trying to carve out your place in the professional world and a prospective client wants a WordPress site, by all means, make them a WordPress site. But expect trouble. And if you ever want to graduate to a saner system, expect to relearn everything.
