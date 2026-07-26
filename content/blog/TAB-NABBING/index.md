---
title: "Tab Nabbing 101"
date: 2026-07-26
draft: false
summary: "Understanding tab-nabbing and reverse tab-nabbing attacks"
tags: ["Phishing"]
categories: ["blog"]
series: ["Social Engineering"]
showToc: true
author: "Moiz Bootwala"
---

Phishing users by changing inactive browser tabs
<!--more-->
In this blog, I'll talk about a very interesting social engineering attack called **Tab Nabbing**, along with its lesser-known sibling, **Reverse Tab Nabbing**. These attacks have the same goal: swapping the trusted tab for a phishing page without the user clicking any suspicious links to get there.

*Both variants are fixed to an extend in modern upto-date browsers.*

## Understanding The Attack

### Tab Nabbing

In tab nabbing attacks, the original tab (the one the user was already on) gets silently rewritten by JavaScript while it's in the background, once the user opens a second tab and stops looking at the first one. 

The attacker takes advantage of a tab the user opened earlier (usually via a link that opens in a new tab or just a tab the user had lying around) and changes its content while the user is away, so that when the user comes back, they believe they are still on the trusted site which they left.

Compared to other phishing attacks, these attacks are harder to detect and have higher chances of success because the contents of the target application is swapped only when the user has opened another tab (which means the user has no visibility). When the user goes back, the swapped content is implicitly trusted.

> *Events like `blur`, `focus`; page visibility APIs like `visibilitychange`, `document.hidden` can be used to facilitate the change on the perfect time. This ensures nothing changes while the user is watches. However the moment tabs are switched, everything changes.*

The below diagram demonstrates how classic tab nabbing works:

![Process of classic tabnabbing](https://cdn.ziomsec.com/tab-nabbing/classic-tabnabbing.webp)

This attack can also be performed with JavaScript completely disabled. We can use `<meta http-equiv="refresh"` element to reload a new page after a time interval. The below example code will wait 5 minutes before swapping the tab with `phish.example.com`

```html
<meta http-equiv="refresh" content="300; url=https://phish.example.com">
```

This technique was named and popularized by Aza Raskin and Eitan Adler in 2010.

### Reverse Tab Nabbing

Reverse tab nabbing happens when a site the user opens in a new tab can control the original tab. It flips the direction of the attack. Instead of the tab a user's on flipping against them, it's a new tab (usually one that was opened by the user from a link on a page they trust) that reaches back and rewrites the original tab the user came from.

In this attack, once the user opens a new tab or window via `target="_blank"` or `window.open()`, the browser hands that new tab a live JavaScript reference back to the page that opened it (which is accessible as `window.opener`). A `Same-Origin` policy stops the new tab from reading anything sensitive out of the opener, but it does not stop the new tab from writing to `window.opener.location` (essentially the url of the tab that opened the new page). Hence, attacker's can use JavaScript in the new tab to silently swap the contents of the trusted tab the user came from.

The below diagram demonstrates how reverse tab nabbing works:

![Process of reverse tab nabbing](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnabbing.webp)

## Root Cause Analysis

Both attacks are made possible from the same underlying assumptions that browsers used to make: a tab's URL can be allowed to be rewritten if a script has a handle to it.

Given below is a code vulnerable to reverse tab nabbing:

```html
<html>
    <body>
        <!-- using HTML to open a new page -->
        <a href="http://evil.example.com" target="_blank"> Attacker Website</a>
        <!-- using JavaScript to open a new page -->
        <button onclick="window.open('https://evil.example.com')"> Attacker Website </button>
    </body>
</html>
```

The `evil.example.com` website would have the following contents:

```html
<html>
    <body>
        <script>
            if (window.opener) {
                window.opener.location = "https://phish.example.com";
            }
        </script>
    </body>
</html>
```

The moment a user clicks either the link or the button above, the new tab opens holding a reference to `window.opener` and the instant that script runs, the inactive tab the user came from gets silently replaced with `https://phish.example.com`.

This works because it doesn't perform anything that a policy like `same-origin` blocks. It is not a cross-origin read. It's a cross-origin **write**. The `evil.example.com` page still can't read the user's cookies or grab anything off their DOM through `window.opener` - `same-origin` blocks that. `.location` is one of the few properties (along with `closed`, `frames`, `length`, `parent`, `self`, `top`) that a cross-origin opener reference is allowed to touch.

> Incase you're not aware about what `cross-origin`, `same-origin policy` is:
> - **Origin** is basically `schema` + `host` + `port`. Two URLs are same-origin only if all three match exactly. Hence, `https://example.com`, `https://www.example.com` and `http://example.com` are all different origins.
> - **Same-Origin Policy** (SOP) is the browser's core sandboxing rule. Scripts running on origin A cannot read data belonging to origin B.
> - **Cross-Origin** just means the two hosts don't share an origin. A cross origin read is always blocked by SOP. A cross-origin write is what Tab Nabbing exploits.

Also, `window.open()` and `<a target="_blank">` are two separate vectors. Browsers fixed the anchor case by making `target="_blank"` imply `rel="noopener"` by default. That fix doesn't carry over to `window.open()` calls; they still hand over a live opener reference unless `noopener` is explicitly passed. 

When we add `rel="noopner"` to a link or `window.open()`'s third argument, the newly opened page's `window.opener` is set to null. Since 2021, the `noopener` property was made the default in `target="_blank"` for `<a>` and `<area>` tags.

However, it's not just `<a>` or `<area>` tags that can be weaponized. A `<form target="_blank">` submission creates the exact same live opener relationship, and it's not covered by the browser-level anchor fix.

Hence, a safe pattern for the `<button>` example is:

```javascript
window.open('https://evil.example.com', '_blank', 'noopener,noreferrer')
```
> Or we can avoid `window.open` for external navigation and use a real anchor with `rel="noopener noreferrer"` instead.

`noreferrer` strips the `Referer` HTTP header (and the `Referrer-Policy`) when navigating to the linked page, so the destination site can't see what page/URL the user came from. `noreferrer` also happens to imply `noopener` behavior as a side effect in most browsers.

Classic tab nabbing's root cause is much simpler and older. For a long time, there was nothing stopping a page from navigating itself on a timer or event, regardless of whether someone was actually looking at it. While this is a legitimate functionality to have (which is why browsers still allow it), not having any technical control makes it a problem. Most of the mitigations for this attack live at the user or process layer.

## Practical Demonstration

Let's look at what this attack looks like in practice. All the files needed for this practical can be found in the following repository:
- https://github.com/RIZZZIOM/tabnabbing-demo

### Lab Layout And Setup

```
tabnabbing-demo/
demo-lab/
├── run-lab.sh                          # starts the 3 servers below
├── trusted-site/
│   ├── vulnerable.html                 # reverse tab nabbing: unfixed
│   └── fixed.html                      # reverse tab nabbing: remediated
├── attacker-site/
│   └── malicious.html                  # the window.opener.location payload
├── phishing-site/
│   └── login.html                      # the fake login clone
└── classic-tabnabbing/
    ├── inactive-tab.html                # JS / visibilitychange variant
    ├── meta-refresh.html                # no-JS <meta refresh> variant
    └── meta-refresh-target.html         # its redirect target
```

1. Clone the repo

```shell
$ git clone https://github.com/RIZZZIOM/tabnabbing-demo.git
```

2. Make the server script executable and run it

```shell
$ chmod +x run-lab.sh
$ ./run-lab.sh
```
> This starts python servers on 4 different ports

### Reverse Tab Nabbing

1. Visit the vulnerable page on `http://127.0.0.1:8000/vulnerable.html`

![visiting the vulnerable page](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnab-1.webp)

This page is a mock version of a community board, the kind of site where any user can post a link. It has two ways to open the "attacker" page, both vulnerable:

```html
<a href="http://127.0.0.1:8001/malicious.html" target="_blank" rel="opener">
  Open shared resource ↗
</a>

<button onclick="window.open('http://127.0.0.1:8001/malicious.html')">
  Open via button (window.open)
</button>
```

The link has `target="_blank"` and `rel="opener"`, and the button calls `window.open()` with just a URL and nothing else. The `rel="opener"` on the link was added because since Firefox 79, Chrome/Edge 88, and Safari 12.1, browsers treat a plain `target="_blank"` link as `noopener` by default, so a link with no `rel` at all would no longer leak `window.opener` on a modern browser. Adding `rel="opener"` manually opts back into the old, insecure behaviour.

Both the link and the button hand the new tab a live `window.opener` reference back to this page with nothing stopping the attacker page from rewriting this tab the moment the user looks away.

2. Click the link or button

![clicking the link or button](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnab-2.webp)

A new tab opens, loading `malicious.html` from the attacker server. This is done using the reference that the new tab holds to the tab we were just on.

3. The `malicious.html` file runs the following script in the background that checks if `window.opener` exists, waits 2 seconds, then rewrites the original tab's location.

```js
if (window.opener) {
  setTimeout(() => {
    window.opener.location = "http://127.0.0.1:8002/login.html";
  }, 2000);
}
```

4. Switch back to the original tab to view the phishing login page. Notice the URL.

![visiting the modified page](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnab-3.webp)

*Video Demonstration:*

![reverse tab nabbing with button demo](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnab-button.gif)

![reverse tab nabbing with link demo](https://cdn.ziomsec.com/tab-nabbing/reverse-tabnab-link.gif)

> When the same actions are repeated on the fixed page hosted on `http://127.0.0.1:8000/fixed.html`, nothing happens.

***What's different in `fixed.html`?***
The `rel="noopener noreferrer"` and the `noopener,noreferrer` window features both set `window.opener` to `null` in the new tab, so `malicious.html`'s `if (window.opener)` check simply fails and nothing runs.

*Video Demonstration:*

![remediated page](https://cdn.ziomsec.com/tab-nabbing/remediated-tabnab.gif)

### Classic Tab Nabbing

1. Open the JS based demo at `http://127.0.0.1:8003/inactive-tab.html`

![visiting the vulnerable page](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-1.webp)

The page has this listener running:
```js
document.addEventListener('visibilitychange', () => {
  if (document.hidden && !hasRewritten) {
    setTimeout(() => {
      hasRewritten = true;
      // rewrite title, favicon, and page content here
    }, 4000);
  }
});
```

It listens for the `visibilitychange` event, and the moment `document.hidden` becomes true, meaning you've switched away from the tab, it starts a 4 second timer.

2. Open a new tab or switch to a different window and wait for 4 seconds.
3. Switch back and notice the contents of the page

![visiting the switched page](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-2.webp)

> All of this happened while you weren't looking at the tab at all. The `hasRewritten` flag just makes sure it only does this once.

*Video Demonstration:*

![demo of classic tab nabbing via js](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-js.gif)

1. Open the no JavaScript version by navigating to `http://127.0.0.1:8003/meta-refresh.html`

![visiting no-javascript page](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-3.webp)

The entire "attack" here is a single line in the page head:

```html
<meta http-equiv="refresh" content="8; url=meta-refresh-target.html">
```

2. After this, switch tabs or windows and after 8 seconds, watch tab-nabbing in action.

![switching to the modified page](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-4.webp)

The page has already redirected itself to `meta-refresh-target.html`, a fake login page, with zero scripting involved. Even a browser with JavaScript fully disabled would still fall for this one.

*Video Demonstration:*

![classic tab nabbing demo](https://cdn.ziomsec.com/tab-nabbing/classic-tabnab-nojs.gif)

## Hardening

The reason this attack works is because there is no browser level restriction stopping one tab, whether it's the tab itself or a new tab it opened, from redirecting a tab that is sitting in the background.

The durable fix for these attacks is *defense in depth*. 
- Always ship `rel="noopener noreferrer"` on `target="_blank"` links 
- Pass `"noopener,noreferrer"` to `window.open()` 
- Add `Cross-Origin-Opener-Policy` headers, and lint with `react/jsx-no-target-blank` 
- Add mitigation at the human layer too, through re-auth prompts, URL checks, and password managers 

Here is what that human layer piece looks like in practice: 
- Don't trust that a tab which "looks" logged in still has a valid session. Verify server side before anything sensitive happens 
- Password managers won't autofill credentials on a spoofed origin, since they check the actual domain and not the page's visual appearance 
- As with any other social engineering attack, awareness plays a huge role in mitigating this one too

### Reverse Tab Nabbing Hardening

In modern browsers, links that use `target="_blank"` now have implicit `rel="noopener"`, so this vulnerability isn't as widespread and critical as before. Also, using `rel="noreferrer"` implies `rel="noopener"`, so if you have chosen to use `rel="noreferrer"`, the use of `rel="noopener"` isn't required.

That said, it is still worth setting those tags explicitly rather than relying on the browser default:

```html
<a href="https://example.com" target="_blank" rel="noopener noreferrer">...</a> 
```

- `window.open()` does not add those properties by default, so you need to pass them yourself:

```javascript
const w = window.open(url, '_blank', 'noopener,noreferrer');
```

## References

- [OWASP - Reverse Tabnabbing](https://owasp.org/www-community/attacks/Reverse_Tabnabbing)
- [OWASP Web Security Guide](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/11-Client-side_Testing/14-Testing_for_Reverse_Tabnabbing)
- [MDN noopener](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel/noopener)
- [MDN noreferrer](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel/noreferrer)

