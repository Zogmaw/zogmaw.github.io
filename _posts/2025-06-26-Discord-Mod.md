---
layout: post
title: Modifying Discord
date: 2025-06-26 10:24 -0500
categories:
- Projects
tags:
- tech
---
# Problem

Discord released a new [update](https://discord.com/blog/get-more-from-your-boosts-with-new-server-perks) recently introducing new server perks.  'Enhanced role styles', however, hurts my eyes and there's no option to disable it.  In the accessability settings, reduced motion can be enabled to remove the 'glow' when hovering over the gradient.  Within the same setting page, role colors can be set to show next to names (or not at all), but this makes it more difficult to identify who people are in my opinion.  It also looks like garbage.  Surely there's a way to just disable the gradient so reading names isn't painful but still keep their color?  Wrong.

# Investigation

## Getting dev access

Luckly for us, discord is an electron app which means it's basically a web browser.  As with any normal webpage, we can open up dev tools and inspect elements to our hearts' content.  Discord did make this a little trickier to do on their desktop client though in the name of security to prevent users from getting phished.  To enable it, we first need to go to our settings.json file at `%appdata%/discord/settings.json` and add a new property `"DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true`.  After a saving the file and restarting the app, we can open up the dev tools with `ctrl+shift+i`.  If you've ever done this in your browser with F12, it'll look very familiar.

## Poking around

The next step was figuring out *what* exactly causes the gradient.  Within the elements tab, we can find out what makes discord look the way it does.  Hovering over the html will highlight the section of the page it relates to.  We could go through and highlight every line of html, but we can do the reverse instead.  The button in the top left of the dev tools window allows you to hover over an element on the page and get directed to its corrosponding html - way better!  So I select that button and click on an annoying name with a bothersome gradient.  The HTML looked something like this:

```HTML
<span class="username_c19a55 desaturateUserColors__41f68 threeColorGradient_e5de78 usernameGradient_e5de78 convenienceGlowGradient_e5de78 clickable_c19a55" aria-expanded="false" data-text="redacted_username" role="button" tabindex="0" style="--custom-gradient-color-1: #a9c9ff; --custom-gradient-color-2: #ffbbec; --custom-gradient-color-3: #ffc3a0; text-decoration-color: rgb(169, 201, 255);">redacted_username</span>
```

Now we're getting somewhere.  It looks like there's a few things mentioning gradient right here.  My first instinct was to investigate the 'style' attribute, since that's what gives an element its look.  I first tried removing all of the --custom-gradient-color variables, but was left with just a blank name.  Even adding a color: property still left me with no name.

After playing around with a few more style combinations and getting nowhere, I remembered that HTML tags (<span>, in this case) can inherrit css from classes.  CSS, if you're unfamiliar, is the styling of webpages and also what's written in the style attributes of HTML tags.  My next step now was to find the css file containing the styles for the classes.  I foolishly did this by going to the sources tab of the dev tools and searching each css file for the classes.  However, within the elements tab you can see the styles of the selected element and where they come from, allowing you to easily find the file.  The classes I was particularly interested in were username, desaturateUserColors, threeColorGradient, usernameGradient, convenienceGlowGradient, and I also found a twoColorGradient class after opening the css file.  Most of them ended up not being particularly interesting, except for (believe it or not) the ones with Gradient in the name.

### usernameGradient

Can make the name just white, no color

### twoColorGradient and threeColorGradient

Change the linear-gradient to just background: var(1) to make it the 1st solid color