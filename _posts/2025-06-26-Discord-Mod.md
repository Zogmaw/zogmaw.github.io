---
layout: post
title: Modifying Discord
date: 2025-06-26 10:24 -0500
categories: [Projects]
tags: [tech]
---
# Problem

Discord released a new [update](https://discord.com/blog/get-more-from-your-boosts-with-new-server-perks) recently introducing new server perks.  'Enhanced role styles', however, hurts my eyes and there's no option to disable it.  In the accessability settings, reduced motion can be enabled to remove the 'glow' when hovering over the gradient.  Within the same setting page, role colors can be set to show next to names (or not at all), but this makes it more difficult to identify who people are in my opinion.  It also looks like garbage.  Surely there's a way to just disable the gradient so reading names isn't painful but still keep their color?  Wrong.

![gradient](/assets/images/gradient.png)
_Username Gradient_

![next-to-name](/assets/images/nexttoname.png)
_Accessability feature showing role color next to name_

# Investigation

## Getting dev access

Luckly for us, discord is an electron app which means it's basically a web browser.  As with any normal webpage, we can open up dev tools and inspect elements to our hearts' content.  Discord did make this a little trickier to do on their desktop client though in the name of security to prevent users from getting phished.  To enable it, we first need to go to our settings.json file at `%appdata%/discord/settings.json` (for windows installs; `~/snap/discord/current/.config/discord/settings.json` for linux) and add a new property `"DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true`.  After a saving the file and restarting the app, we can open up the dev tools with `ctrl+shift+i`.  If you've ever done this in your browser with F12, it'll look very familiar.

## Poking around

The next step was figuring out *what* exactly causes the gradient.  Within the elements tab, we can find out what makes discord look the way it does.  Hovering over the html will highlight the section of the page it relates to.  We could go through and highlight every line of html, but we can do the reverse instead.  The button in the top left of the dev tools window allows you to hover over an element on the page and get directed to its corrosponding html - way better!  So I select that button and click on an annoying name with a bothersome gradient.  The HTML looked something like this:

```HTML
<span class="username_c19a55 desaturateUserColors__41f68 threeColorGradient_e5de78 usernameGradient_e5de78 convenienceGlowGradient_e5de78 clickable_c19a55" aria-expanded="false" data-text="redacted_username" role="button" tabindex="0" style="--custom-gradient-color-1: #a9c9ff; --custom-gradient-color-2: #ffbbec; --custom-gradient-color-3: #ffc3a0; text-decoration-color: rgb(169, 201, 255);">redacted_username</span>
```

Now we're getting somewhere.  It looks like there's a few things mentioning gradient right here.  My first instinct was to investigate the 'style' attribute, since that's what gives an element its look.  I first tried removing all of the --custom-gradient-color variables, but was left with just a blank name.  Even adding a color: property still left me with no name.

![blank-name](/assets/images/blankname.png)
_Username after removing custom-gradient-color variables_

After playing around with a few more style combinations and getting nowhere, I remembered that HTML tags (<span>, in this case) can inherrit css from classes.  CSS, if you're unfamiliar, is the styling of webpages and also what's written in the style attributes of HTML tags.  My next step now was to find the css file containing the styles for the classes.  I foolishly did this by going to the sources tab of the dev tools and searching each css file for the classes.  However, within the elements tab you can see the styles of the selected element and where they come from, allowing you to easily find the file.  The classes I was particularly interested in were username, desaturateUserColors, threeColorGradient, usernameGradient, convenienceGlowGradient, and I also found a twoColorGradient class after opening the css file.  Most of them ended up not being particularly interesting, except for (believe it or not) the ones with Gradient in the name.

### usernameGradient

This is defined as follows in the css file:
```CSS
.usernameGradient_e5de78 {
    -webkit-background-clip: text;
    background-clip: text;
    background-size: 100px auto;
    -webkit-text-fill-color: transparent
}
```
Playing around with these, adding a color property didn't do anything.  However, removing the -webkit-text-fill-color property turned all names to white.  While this does get rid of the gradient, I could just remove the role colors in the options.  I wasn't able to keep the role color using this class though.


### twoColorGradient and threeColorGradient

These classes were defined as follows:

```CSS
.twoColorGradient_e5de78 {
    background: linear-gradient(to right,var(--custom-gradient-color-1),var(--custom-gradient-color-2),var(--custom-gradient-color-1))
}

.threeColorGradient_e5de78 {
    background: linear-gradient(to right,var(--custom-gradient-color-1),var(--custom-gradient-color-2),var(--custom-gradient-color-3),var(--custom-gradient-color-1))
}
```
Some usernames use the two color gradient while others use the three, probably depending on how the role is configured.  Changing the background property in these classes changes the color of the usernames that use the class.  Now we're getting somewhere!  In order to preserve some relation between color and role, we don't want to just set the background to a static color.  Based on the linear-gradient function call, it looks like there are colors stored in variables.  Changing the background property to just `var(--custom-gradient-color-1)` achieved the desired result.


## Automation

It would be a bit tedious to need to comb through css files to find the correct one with these classes then edit the background property every time I launch discord.  The easiest was to do this is probably with some javascript, so I started doing some more research.  Here was my basic psudocode

1. Find the css file with the color gradient classes
2. Find the color gradient classes within the css file
3. Replace the background attribute with just the first color variable
4. Profit

Thankfully there are a ton of builtin javascript methods to make this easy.  `document.stylesheets` [returns a list of all the associated stylesheets](https://developer.mozilla.org/en-US/docs/Web/API/Document/styleSheets) on a page, listed first by linked headers in header order, then from the DOM in tree order.  This is great, especially since it's pre-sorted.  Running this in the console returns 18 stylesheets.  We know the name of the css file we need, so looking at the href property shows us that the first list entry is what we need (at index 0).

Now that we have the css file, we need to find the gradient classes within it.  The rules property has a list of CSSRules, which we can iterate through.  A sample CSSRule has the following properties

```JavaScript
cssRules: CSSRuleList
cssText: ".twoColorGradient_e5de78 { background: linear-gradient(to right,var(--custom-gradient-color-1),var(--custom-gradient-color-2),var(--custom-gradient-color-1)); }"
parentRule: null
parentStyleSheet: CSSStyleSheet {ownerRule: null, cssRules: CSSRuleList, rules: CSSRuleList, type: 'text/css', href: 'https://discord.com/assets/12633.675d62620d1094b7.css', …}
selectorText: ".twoColorGradient_e5de78"
style: CSSStyleDeclaration {0: 'background-image', 1: 'background-position-x', 2: 'background-position-y', 3: 'background-size', 4: 'background-repeat', 5: 'background-attachment', 6: 'background-origin', 7: 'background-clip', 8: 'background-color', accentColor: '', additiveSymbols: '', alignContent: '', alignItems: '', alignSelf: '', …}
styleMap: StylePropertyMap {size: 9}
type: 1
```

Looking at the properties of the rules, most have the 'selectorText' property, which appears to be the class name.  We can look for the required classes using this property.  In order to match on the classes we want, we can use the `includes` function of strings to search for a substring within them.  For example, `"testing".includes("tin")` would return `true`.

With that in mind, we can run the following code in our console to find where the gradient classes are within out list

```JavaScript
for (let i = 0; i< document.styleSheets[0].rules.length; i++) {
    let txt = document.styleSheets[0].rules[i].selectorText
    if (txt != undefined) {
        if(document.styleSheets[0].rules[i].selectorText.includes("ColorGradient")) {
            console.log(i)
        }
    }
}
```

The above code will print the indicies at which the gradient classes are in the list.  Now that we've found them, the next step is to modify their styles to only use the first color in the gradient.  We can make a simple change to our code to bring us to our final result:

```JavaScript
for (let i = 0; i< document.styleSheets[0].rules.length; i++) {
    let txt = document.styleSheets[0].rules[i].selectorText
    if (txt != undefined) {
        if(document.styleSheets[0].rules[i].selectorText.includes("ColorGradient")) {
            console.log(i)
            document.styleSheets[0].rules[i].style.background = "var(--custom-gradient-color-1)"
        }
    }
}
```

After running the script, we can see usernames are a solid color again.  Success!

![good-names](/assets/images/fixedUsernames.png)
_Usernames are once again one color_


# Future Work

This isn't a bad solution, but it does require opening the dev console and copy/pasting this command every time we start discord.  I'm not sure how to further automate this, but it's something I'd like to look into.

There is also another semi-related feature with the gradients that has some type of glow when hovering over a name.  This doesn't bother me nearly as much so I decided to leave it in, but if it does in the future it can likely be 'disabled' in a similar way.