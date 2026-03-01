---
title: Earn Bing Rewards points on Linux using non-Edge Chromium browsers
date: 2026-03-01 11:00:53 +0100
categories: [Linux]
tags: [linux]
---

## Context

According to the [Chrome for developers documentation](https://developer.chrome.com/docs/extensions/reference/manifest/chrome-settings-override):

> Settings overrides are a way for extensions to override selected Chrome settings. The API is available **on Windows and Mac** in all current versions of Chrome.

Because settings overrides are unavailable on Linux, the Bing Rewards Chrome extension can't set the correct search engine URL that allows you to earn points. However, we can easily work around this by extracting the URL from the extension files and adding it as a custom search engine. You can even throw away the extension and its telemetry after doing this!

## How To

1. Download a Chrome extension that allows you to view the `.crx` (source code) of the Bing Rewards extension, such as [Chrome extension source viewer](https://chromewebstore.google.com/detail/chrome-extension-source-v/jifpbeccnghkjeaalbbjmodiffmgedin). Online services that do this also exist, but I haven't had any luck with them.

2. Once installed, navigate to the [Microsoft Bing Search with Rewards extension](https://chromewebstore.google.com/detail/microsoft-bing-search-wit/fbgcedjacmlbgleddnoacbnijgmiolem) page in your web browser. Right click anywhere on the web page, and click on "View extension source".

3. Navigate to `manifest.json` and copy the value of "search_url", ending at `&q=`. As of March 2026, the correct fragment to copy is `https://www.bing.com/search?FORM=U523MF&PC=__PARAM__U523&q=`. You could've simply copied the URL from here, but that way you'd only be [fed for a day](https://en.wiktionary.org/wiki/give_a_man_a_fish_and_you_feed_him_for_a_day;_teach_a_man_to_fish_and_you_feed_him_for_a_lifetime) ;)

4. Now it's time to add the search URL. Open your browser Settings, navigate to "Search engine", then "Manage search engines and site search". Scroll down to the bottom and add a new site search engine.
    - For **Name**, use `Bing`. You can actually set this to whatever.
    - For **Shortcut**, anything is okay as well; I suggest `:bi`. `:b` may not be accepted since it's already used for the default Bing.
    - **URL with %s in place of query** is the important part. Paste the URL found in `manifest.json` here. Remember to add `%s` after `&q=` - this lets the browser know where to insert your search query into the URL.

5. Click on the three dots next to your new custom search engine, and make it the default.
