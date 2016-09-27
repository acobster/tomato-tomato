{:title "Debugging JavaScript Performance Issues in a Legacy Codebase"
 :layout :post
 :tags  ["javascript performance debugging"]}

## or, How Chrome Dev Tools Helped Me Do My Job

It's not every day that I debug JavaScript performance, and typically I'm able to locate the issue with some increasingly focused `console.log` calls. This time, I had to dive a little deeper and learn how to use Chrome's Profiles tool. It is awesome!

The problem I'm trying to solve today is your standard, vague performance issue: *why is this page soooo slow?*

## On Technical Debt

I can't tell you much about this code because reasons, but I can tell you that we inherited a lot of technical debt from the previous devs and the code is really repetitive to the point of being *extremely* brittle. This page we're loading...well, it's actually a bunch of different pages and we switch between them using JavaScript. So all the content is loaded at once. It's pretty terrible. I'm only telling you this because I say some stuff later that might make me sound like a total n00b if you didn't know it's because technical debt, okay? Anyway.

## Let's Start Debugging

I load up the Timeline tab in Chrome Dev Tools. This is a tool that shows you what your page is spending its time doing, and how much of it. And...yikes.

![Dev Tools Timeline](/img/dev-tools-timeline.png)

Almost *eleven seconds* to completely render the page, over five of which are spent scripting. I record a few more page loads and average the highest numbers:

* Scripting: 6293.2 ms
* Other: 1280.8 ms
* Loading: 797.68 ms

There's also a decent chunk of "idle" time, which according to some [really thorough internet research](http://stackoverflow.com/questions/33769070/chrome-dev-tools-timeline-white-bars#33883219) is GPU-bound. I guess that's things like rendering fancy CSS? That's useful information, but there's not much I can do on that front today. Remember how I mentioned technical debt? Yeah.

So we're averaging a little over six seconds running JavaScript, almost five times as long as the next biggest offender. Let's find out why.

Next, I load up the Profile tab. After selecting the **Record JavaScript CPU Profile** option, I click **Start** and click through some of the particularly slow features on the offending page. I get annoyed, because it's *really slow.*

After accumulating some detail on what the JS is doing I click **Stop.** I open up the Profile that Dev Tools has just created for me and find a bunch of function calls, logged in order of time they took! This is exactly what I'm looking for.

Over on the left, it has a couple different columns for **Self Time** and **Total Time.** They don't look that different for the most part, but sometimes Total Time is much bigger. I wonder why? According to some more [really thorough internet research](http://stackoverflow.com/questions/7127671/difference-between-self-and-total-in-chrome-cpu-profile-of-js#7127705), the Self Time for a function call means how much time the CPU spent crunching numbers for code *just inside that function* before it either delegated a task to another function or returned. So a big Self Time means "this is the function you're looking for" while a big Total Time and comparably small Self Time means "dig into the call stack here!"

Anyway, the CPU Profile has some interesting findings:

![Wistia Video's CPU Usage at All-Time High](/img/dev-tools-cpu-profile.png)

By the way, I left out the `(program)` part of the CPU Profile report, because that's Chrome itself running. That's not something you're generally very likely to have much control over

Here you can see that the top two JS function calls take up about 2.7 seconds: almost half of the average time our page spent scripting. It even shows us where each function is defined. In this case our big players and most of the littler players, too, are in `https://fast.wistia.com/assets/external/E-v1.js`. Oh, great. Typical third-party tool that you can't do anything about!

Wistia is a video platform with a pretty sweet UI and built-in analytics and probably some other cool stuff. But in this case I am mad at it because it is hogging my CPU! What can I do?

Remember how I said the one page is actually a bunch of pages? Let's refactor our JavaScript so that it only loads the slow Wistia stuff when it really needs to, i.e. when we're on, I dunno, a video page? I won't bore you with the details, mostly because I had to take bad code and make it a little worse with this optimization. Here's how we fare after the optimization:

![No more eager Wistia code!](/img/dev-tools-cpu-profile-faster.png)

Much better! The five longest running functions together take less than a tenth of a second!

Let's see how our Timeline looks now...

![But why is it slower?!](/img/dev-tools-timeline-slower-wut.png)

What?

