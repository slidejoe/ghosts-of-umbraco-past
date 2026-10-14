---
theme: slidev-theme-rockdove
title: Ghosts of Umbraco Past
info: |
  ## Ghosts of Umbraco Past
  Patterns that were correct advice at the time — and what changed underneath.
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: fade
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# https://sli.dev/features/drawing
drawings:
  persist: false
layout: intro
---

# Who'd merge this?

```csharp
var articleList = UmbracoContext.Content.GetAtRoot()
    .DescendantsOrSelf<ArticleList>().FirstOrDefault();

var allArticles = articleList.Children<Article>()
    .Where(x => x.IsVisible())
    .OrderByDescending(x => x.ArticleDate);

var pageOfArticles = allArticles.Skip((pageNumber - 1) * pageSize).Take(pageSize);
var totalItemCount = allArticles.Count();
```

<!--
You've had a few minutes to look at this — **can I get
  a read of the room? Who'd merge this?**

And who wouldn't?

It *was* correct - **every hand that went up was right — for the Umbraco they learned on.**
-->

---
layout: cover
---

# Ghosts of Umbraco Past

Coding Practices to Lay to Rest in 2027

<!--
We're here to look at practices and patterns we've picked up or been taught over the years that are no longer the best way to do things. **Every pattern in this talk was correct advice at the time.** But ideas and technology have moved on.

We'll look at **three places these ghosts haunt: your code, your editors, and your infrastructure.** And **we're not here to judge** - there are a lot of moving parts to keep track of and learned habits are hard to change. I strongly suspect that everyone in this room, myself included, is still doing some of these, and at the very least has some of these patterns in a codebase. All the examples are real; where they're contrived, it's only to anonymise clients. And we'll end on some advice for how to spot these lapsed patterns in the future.
-->

---
layout: bio
tagline: Senior Developer at Bump
image: /images/joe-portrait.png


---

- <logos-mastodon-icon /> [@joe@umbraco&#8239;community.social](https://umbracocommunity.social/@joe)
- <logos-bluesky /> [@joe.gl](https://bsky.app/profile/joe.gl)
- <solar-global-bold-duotone /> [joe.gl/ombek](https://joe.gl/ombek)

<!--
My name is Joe Glombek, I'm an **Umbraco Master and MVP** and have been working with Umbraco for **15 years** now!

I'm in a unique position working at Bump: **we work inside other agencies and get dropped in to solve really specific performance issues** - so we see a lot of these legacy patterns and can identify the ones that come up over and over.

Since we're telling ghost stories today, this is a photo of me being scared by a ghost in a top hat at the house of horrors in Tivoli.
-->

---
layout: intro
# ghostly fade between sections

---

# Haunting your code

<!--
In this section, we'll cover **patterns in the code we write that used to be performant, but the ground moved under them.** The underlying platform changed - be that Umbraco, .NET or C# - meaning the original pattern is less efficient or we now have new, more performant or easier to read options available.

These patterns are often found in old codebases that have been upgraded through versions, so they're **ones to watch when you're doing your own upgrades.**
-->

---
layout: two-cols-header
---

# Traversing the Tree
<small>The ghost of NuCache (2024, v14)</small>

::left::

```csharp{all|2|3|7|11-12|all}
var articleList = UmbracoContext.Content
  .GetAtRoot()
  .DescendantsOrSelf<ArticleList>()
  .FirstOrDefault();

var allArticles = articleList
  .Children<Article>()
  .Where(x => x.IsVisible())
  .OrderByDescending(x => x.ArticleDate);

var pageOfArticles = allArticles
  .Skip((pageNumber - 1) * pageSize).Take(pageSize);

var totalItemCount = allArticles.Count();
```

::right::

<!-- Invisible click step (was the "After" heading): the code below is timed against it -->
<v-click><span /></v-click>

<div v-click="['-1', '+2']">

````md magic-move
```csharp
var articleList = Umbraco.ContentAtRoot()
  .FirstOrDefault()?.FirstChild<ArticleList>();

if (!_examineManager.TryGetIndex("ExternalIndex", out IIndex? index))
return;

var results = index.Searcher.CreateQuery("content")
    .NodeTypeAlias(Article.ModelTypeAlias)
    .And()
    .ParentId(articleList.Id) // Or Search Extensions with .Path()
    .OrderByDescending(new SortableField("articleDate",
      SortType.Long))
    .Execute(new QueryOptions((pageNumber - 1) * pageSize, pageSize));

var pageOfArticles = results
    .ToPublishedSearchResults(UmbracoContext.Content)
    .Select(r => r.Content)
    .OfType<Article>();

var totalItemCount = results.TotalItemCount;
```

```csharp
if (!_examineManager.TryGetIndex("ExternalIndex", out IIndex? index))
return;

var results = index.Searcher.CreateQuery("content")
    .NodeTypeAlias(Article.ModelTypeAlias)
    .And()
    .ParentId(articleList.Id) // Or Search Extensions with .Path()
    .OrderByDescending(new SortableField("articleDate",
      SortType.Long))
    .Execute(new QueryOptions((pageNumber - 1) * pageSize, pageSize));

var pageOfArticles = results
    .ToPublishedSearchResults(UmbracoContext.Content)
    .Select(r => r.Content)
    .OfType<Article>();

var totalItemCount = results.TotalItemCount;
```
````
</div>

<div v-click="['+0', '+1']">

```csharp
var results = await _searcher.SearchAsync(
  indexAlias: Constants.IndexAliases.PublishedContent,
  filters: [
    new KeywordFilter(
      FieldName: Constants.FieldNames.ParentId,
      Values: [articleList.Key.ToString()],
      false
    )
  ],
  sorters: [ new DateTimeOffsetSorter("articleDate", Direction.Descending) ],
  skip: (pageNumber - 1) * pageSize, take: pageSize);

var pageOfArticles = results.Documents
  .Select(
    d => UmbracoContext.Content.GetById(d.Id))
  .OfType<Article>();

var totalItemCount = results.Total;
```

</div>

<div v-click="'+0'">

```json
"Umbraco": {
  "CMS": {
    "Cache": {
      "ContentTypeKeys": [ "<Article doc type key>" ],
      "DocumentBreadthFirstSeedCount": 500
    }
  }
}
```

</div>

<!--
This is the example from my opening slide - but now we can dive into it!

Here's a pretty typical tree-traversal pattern we've all seen before - in fact this sample is from **an older version of Clean Starter Kit.**

We're getting a root node [Click] and walking the entire tree [Click] to find a listing page, [Click] and then paginating its children [Click].

[Click] This is fine for a small site, and particularly **when the whole tree is cached in memory, as it was up to v14.** This pattern also allowed the starter kit to keep working even if people moved pages about. Starter kits and demos, however, tend to propagate out to larger sites as a "best practice" - **this *exact* snippet is in 20 public repos**, let alone slight variations and private repos.

But even then it had two small costs: **we walk far more of the tree than we need to find one page, and we load every article just to keep ten.**

Those two small costs are exaggerated by changes to the cache in v15: **the content cache is now lazy-loaded**, so every node we touch that isn't already cached gets loaded from the database. A traversal can materialise hundreds of documents, then throw away all but ten. And DescendantsOrSelf is depth-first: right down to the bottom of each branch before it moves across, so **every node before the one you want gets loaded.**

We've got a few options here, but **all of them involve switching from tree traversal to search.** [Click] Search is a much more efficient way of sorting and filtering documents like this.

We've also got a slightly more efficient way of finding the parent node, by traversing the tree more precisely using Children rather than Descendants.

In fact, we don't need to specify the parent node at all - it was a little redundant in the first example - [Click] so we can skip that altogether and just search by page type.

You'll notice we're having to manually convert the IDs the search index gives us to IPublishedContent. This may seem less efficient, but remember **we're now only getting the published content for the 10 IDs we care about - and that's what the cache is fast at!**

But of course, this talk is all about features changing around us, so of course I should be writing this with **the new Umbraco Search abstraction - currently a package, and planned to ship in core with v19 in December.** We can do the same sort of thing here. [Click]

[Click] And a quick win if you can't fix the traversal yet: **cache seeding** - **what's preloaded into the cache at startup.** If you know a listing page is going to walk 200 articles, **ContentTypeKeys preloads every document of the types you list**, however deep they are in the tree, so those reads come from memory from the very first request. And **DocumentBreadthFirstSeedCount preloads the first N documents from the root down** - your home page, landing pages and navigation. It's **100 by default, which is a round number, not a reasoned one, so size it for your site.** It only helps after a cold start, though - **it's a stopgap while you move to search, not the fix.**
-->

---
layout: two-cols-header
---

# `.ToList()` everywhere
<small>The ghost of `IEnumerable<T>` everywhere (2023, C# 11)</small>

::left::

```csharp [NewsService.cs]{all|all|3|all} 
public IEnumerable<IPublishedContent> GetPagedNews(int rootNodeId, ...) {
  // ...
  return results.ToList();
}
```

```csharp [News.cshtml]{all|6|6|all}{at:'-2'}
var items = new List<IPublishedContent>();
if (block.Content.NewsItems == null)
{
    items = _newsService
        .GetPagedNews(homePage.Id, ...)
        .ToList();
}
else
{
    foreach (var item in block.Content.NewsItems)
    {
        items.AddRange(...);
    }
}
```

::right::

<v-click>

```csharp [NewsService.cs]
public IReadOnlyList<IPublishedContent> GetPagedNews(int rootNodeId, ...) {
  // ...
  return results.ToArray();
}
```
</v-click>
<v-click>

```csharp [News.cshtml]
IReadOnlyList<IPublishedContent> items =
  block.Content.NewsItems is null
    ? _newsService.GetPagedNews(homePage.Id, ...)
    : [.. block.Content.NewsItems.SelectMany(...)];
```

</v-click>

<!--
Here's some sample code from a real repo of ours. It's a bit mean of me to pick on this, but **we're actually ToList-ing [Click] a List [Click] here!**

[Click] The service builds a list, then returns it as an IEnumerable, so the caller can't tell it's already been materialised. The developer wants to make sure it isn't lazy-loaded, and the else statement needs a List, so it calls .ToList() again - and **ToList always builds a full copy, even of something that's already a list.**

This often happens because **someone got bitten by multiple enumeration once, and .ToList() became a reflex rather than a decision.** And because it looks like good hygiene, it gets missed in code review.

The reason this pattern is bad now is that we've been given many more interfaces to use - not everything has to be an IEnumerable any more. **Returning an IEnumerable is a lie to the caller: it says "I'm lazy, be careful" when the opposite is true.**

So rather than saying it's an enumerable, we can say it's been materialised, by changing the return type of GetPagedNews [Click] to IReadOnlyList. **The return type is a promise about cost.** IReadOnlyList says "this work's done, enumerate as often as you like". IEnumerable says "this might not have happened yet". And we don't need the overhead of a List here, so it can be an array under the covers.

[Click] We can also replace the logic in the caller so it doesn't need a List at all, using some LINQ and a spread - more on spreads in a couple of slides.

I should be clear: **a List is still the right tool when you're accumulating. The ghost was never List, it was the signature that hid it.**
-->

---
layout: two-cols-header
---

# `DateTime` is so `DateTime.AddYears(-1)`
<small>The ghost of `DateTime.Now` (2023, .NET 7)</small>

::left::

```csharp
public class Offer
{
    public DateTime StartsAt { get; set; }
    public DateTime EndsAt { get; set; }
}

public class OfferService
{
    public bool IsLive(Offer offer)
    {
        var now = DateTime.Now;
        return offer.StartsAt <= now
            && offer.EndsAt > now;
    }
}
```

::right::

<v-click>

```csharp
public class Offer
{
    public DateTimeOffset StartsAt { get; set; }
    public DateTimeOffset EndsAt { get; set; }
}

public class OfferService(TimeProvider time)
{
    public bool IsLive(Offer offer)
    {
        var now = time.GetUtcNow();
        return offer.StartsAt <= now
            && offer.EndsAt > now;
    }
}
```
</v-click>

<v-click>

```csharp [OfferServiceTests.cs]
var time = new FakeTimeProvider(
    new DateTimeOffset(2026, 3, 29, 1, 30, 0,
                       TimeSpan.Zero));
```

</v-click>

<v-click>

⚠️ Umbraco's own dates stay `DateTime` (UTC)

</v-click>

<!--
**DateTime is so last year.** There are actually two ghosts on the left here: **the type, and the static call to DateTime.Now.** I'm sure we've all written both - DateTime.Now and UtcNow are what we've been taught to use for a long time.

This is a contrived example, but it's closely based on real code of ours, and importantly it's our own model, not an Umbraco field. This offer runs from midnight to midnight - but **whose midnight? The editor's? The server's? The visitor's?** DateTime can't tell you, because **DateTime carries no offset.** There's a Kind property, but it quietly doesn't save you: comparing different Kinds compiles fine and does the wrong thing.

[Click] So what should we be doing instead? **DateTimeOffset for any absolute point in time.** Keep DateTime for genuinely abstract dates, like a birthday - you celebrate it on the same day whatever time zone you're in. And instead of the static DateTime.Now, **inject a TimeProvider**, which has been in .NET since version 8.

[Click] Because **DateTime.Now is untestable.** The only way to test what happens just before the clocks change was to change your dev machine's clock - and remember to change it back afterwards. With a TimeProvider, we **inject a FakeTimeProvider in our tests.** This one is set to 1:30am on the 29th of March - an hour that doesn't exist in the UK. **DateTime didn't get worse. Our expectations of testability went up, and .NET finally gave us the tool for the job.**

[Click] One caveat: **Umbraco's own system dates - created, updated, published - are still DateTime.** Since v17 they're always stored in UTC - they didn't used to be - so compare them in UTC, and **never against DateTime.Now.** Your property values can be different: v17's new Date Time with Time Zone editor gives you a DateTimeOffset.
-->

---
layout: two-cols-header
---

# `Union` and `Concat`
<small>The ghost of LINQ glue (2023, C# 11)</small>

::left::

```csharp
var all = featured
    .Union(latest)
    .Concat(new[] { pinnedArticle })
    .ToArray();
```

::right::

<v-click>

```csharp
List<Article> all =
    [.. featured, .. latest, pinnedArticle];
```

</v-click>

<!--
Our next ghost is a quick one. **Does anyone know the difference between Union and Concat?**

**Union de-duplicates as it goes. Concat just sticks the two together.** People reach for Union meaning "and also", and that's a bug.

And it doesn't even de-duplicate reliably: nothing in Umbraco's content models overrides Equals, so **Union is comparing references.** Most of the time the cache hands you the same instance for the same node, but not in preview, and not across a publish. **If you mean de-duplication, say so: DistinctBy on the Key.** Either way it's unclear what's happening, and it's easy to end up with a Union and a Distinct on the end, just to be sure.

[Click] So what I'm proposing instead is **collection expressions. The intent is unambiguous, and the compiler picks the shape.** Here I've made a List, but I could have made it an IReadOnlyList like on the last-but-one slide, and the compiler would create a read-only collection for me. And you'll notice **the spread in C# is two dots, not three like JavaScript.** Don't ask me why. I get it wrong every time.
-->

---
layout: intro
# ghostly fade between sections

---

# Haunting your editors

<!--
The next section of ghosts are the ones that haunt your editors. Here, it's not so much that the technology moved on. **Our expectations moved on, and we've been given new tools to improve the editor experience.**

A lot of what we built was **developer-friendly, but editor-hostile.** We optimised for the person who reads the C#, not the person who uses the backoffice every day.
-->

---
layout: two-cols-header
---

# Block names and labels
<small>The ghost of AngularJS label templates (2024, v13)</small>

::left::

<img src="/screenshots/before-about-us-blocks-editor.png" class="rounded shadow-lg" />

::right::

<div v-click="['+1', '+1']">

<img src="/screenshots/after-about-us-blocks-editor.png" class="rounded shadow-lg" />

</div>

<div v-click="'+0'">

  <img src="/screenshots/add-new-block.png" class="rounded shadow-lg" />

</div>

<!--
First up: block names and labels. I'm sure we've all seen something like this. **A block list showing the element type's name, over and over again.** It's not particularly editor-friendly - imagine trying to find the one item you saw on the front end in here.

[Click] **Block labels let us pull in the value of the heading field**, or loads of other useful things. And we can go further: this last text and image block doesn't actually have any text in it, so **we can say that** - "Text and image (empty)" - rather than just falling back to the type name. Similarly, I've given the related pages block a label, because I just wanted it to say "Related pages".

[Click] But **the document type name is still what editors see when they click Add content.** There's no way to override it in the block config, so "Call To Action Block List Item" is what they pick from, every time. This goes against what a lot of people believe deep down, but my suggestion is: **wherever the name is visible to the editor, make it editor-friendly. The alias is only used in code** - keep it as callToActionBlockListItem if you really want. **Names for editors, aliases for code**, and use folders in your document types to show which ones are block items.
-->

---
layout: two-cols-header
---

# Document type inheritance
<small>The ghost of master document types (2014, v7.1)</small>

::left::

<img src="/screenshots/before-tree-inheritance.png" class="w-75 rounded shadow-lg" />

::right::

<v-click>

<img src="/screenshots/after-tree-compositions-pages.png" class="w-68 rounded shadow-lg" />

</v-click>

<!--
Now I'm sure we've all seen this: **document type inheritance**, five levels deep. We did it because **it felt like object-oriented programming**, and object-oriented was how we thought about everything else in the project - so why wouldn't it be how we think about content types? Lots of older Umbraco sites have this structure somewhere, and it's really difficult to unpick. You can, with some uSync wizardry, but these things tend to last.

The trouble is that **a type can only be in one hierarchy.** I can't inherit my news article page from both an article page and a sponsored page. And **whatever's above you, you get**: if News Article Page needs an author field, then Sponsored News Article Page gets one too, whether it needs it or not.

[Click] What we do now is **use compositions - a document type can use as many as it needs.** I'm a big fan of a Has or Is prefix for compositions. You don't have to use that one, but **some kind of naming convention really helps**, and so does putting them in their own folder. **Inheritance is almost never the right answer now we have compositions.**
-->

---
layout: two-cols-header


---

# The page-builder reflex
<small>The ghost of the Grid Layout (2024, v13)</small>

::left::

<img src="/screenshots/before-summer-campaign-editor.png" class="max-h-90 rounded shadow-lg" />

::right::

<v-click>

<img src="/screenshots/after-summer-campaign-editor.png" class="rounded shadow-lg" />

</v-click>

<!--
The page-builder reflex. Now, I'm not saying page builders are wrong - **there's a place for them. The warning sign is when it's the default for every page.** And editors asked for this freedom for real reasons: rigid schemas, and waiting weeks for a developer to add a field.

My biggest objection is that **there's no data structure here, just layout.** I recently had a client with **the history of a boat** - where it had been, who'd been on board - all in a very page-builder-y layout. Then they designed a lovely interactive, filterable history: who was on board during this time frame? That needs structured data. So I ended up writing an exporter to get from this free-form, put-the-text-wherever-you-want layout into structured data the search could ingest.

**I always used to say I want the data to be pure, because it might not be a website one day.** For years that was very hypothetical. **Now it's real**: a headless front end, a native app, an LLM pointed at your CMS. A CMS that just mirrors one website doesn't necessarily make sense any more. And if a different agency builds a headless front end, a rebuild might leave the CMS where it is and only change the front end - and if the data isn't in a logical structure, that gets very obtuse.

Editors like flexibility, absolutely. But **too much flexibility means entering the same data too many times, in too many places.**

[Click] So this is what I propose. **We work in bands: just blocks, and those blocks are structured.** Any block can still go anywhere, so there's flexibility in the arrangement, but each one is structured data - if we needed to, we could pull that JSON out and move it somewhere else. Some pages genuinely are canvases, and that's fine - **as a decision, not the default.**
-->

---
layout: intro

# ghostly fade between sections
---

# Haunting your infrastructure

<!--
Our final section is the ghosts that haunt your infrastructure. **The ground moved around the code this time: the edge, the build tooling, the browser, and the cost of writing code yourself.** The old plumbing still works - it's just been outclassed.
-->

---
layout: intro
# ghostly fade between sections

---

# myhotel.com/nyc

<!--
Imagine this is **a billboard** for one of our clients. They've just opened a hotel in New York, so it says myhotel.com/nyc.

**/nyc isn't a page. It's a redirect** to /hotels/new-york-city - which is a perfectly fine thing to do. But nobody told us it was going on a billboard. Or on television.

**And it brought the site down.** Let's take a look at why.
-->

---
layout: two-cols-header
---

# Redirects and 404s
<small>The ghost of the ContentFinder chain (2023, v11)</small>

::left::

```csharp
public class LegacyUrlContentFinder
    : IContentFinder
{
    public async Task<bool> TryFindContent(
        IPublishedRequestBuilder request)
    {
        var match = await _db.FetchAsync(
            request.Uri.AbsolutePath);
        ...
    }
}
// stacked behind three more finders,
// each doing the same
```

::right::

<!-- Step 1: rewrite middleware, before Umbraco. Replaced by the edge version on the next click. -->
<div v-click="['+1', '+2']">

```xml[redirects.xml]
<rewrite>
  <rules>
    <rule name="NYC billboard" stopProcessing="true">
      <match url="^nyc/?$" />
      <action type="Redirect"
        url="/hotels/new-york-city" />
    </rule>
  </rules>
</rewrite>
```

</div>

<div v-click="['-1', '+1']">

```csharp[Program.cs]
app
.UseRewriter(new RewriteOptions()
  .AddIISUrlRewrite(
    app.Environment.ContentRootFileProvider,
    "redirects.xml"))
  .UseUmbraco();
```

</div>

<div v-click="'+0'">

```js[redirects-worker.js]
// Cloudflare Worker — never reaches origin
const table = await REDIRECTS.get(
  url.pathname, 'json');

if (table) {
  return Response.redirect(
    table.destination, table.status);
}
```

Table pulled from the Delivery API at build time

</div>

<!--
Here's **a content finder doing redirects**, at the end of the chain. It's roughly how Umbraco's own redirect tracking and Skybrud Redirects work, and it's a pattern you may well have written yourself. **Every lookup is a database query, and none of them are cached** - Umbraco's own finder can make two per unmatched request. And because it runs last, **we go through the whole Umbraco request pipeline before we finally give up and redirect.**

There's a reason it runs last: **if someone creates a page at a path that used to redirect, the page should win.** But for high-volume URLs we *know* are redirects, that's the wrong end of the pipeline. And the 404 path runs for every bot and scanner too, so redirects can be hit more than you expect and can even bring a site down.

So instead of it being the last thing we do, what if it's the first thing we do, we need to bring the redirect earlier in the pipeline. [Click] If you're hosting on IIS, your old web.config rewrite rules still work - and they run before .NET even sees the request. If you're not, the same rules can run as **middleware** [Click] from a redirects.xml, before Umbraco gets involved.

But how about if we do the redirect even earlier, **before we even reach the server?** [Click] We pull all the redirects out of Umbraco automatically using the Delivery API, push them into Cloudflare Workers Table, and have a Worker check every request before anything else. **It's still content-managed, but it's at the edge.** And suddenly **/nyc becomes the fastest request on the site, not the slowest.**

We do the same for malicious URLs. If someone's asking for /wp-admin, we already know it's going to 404 - so **404 at the edge.** It doesn't even need to be a pretty 404.
-->

---
layout: two-cols-header
---

# Bundling
<small>The ghost of server-side bundling (2024, v13)</small>

::left::

```html
<script src="~/bundles/main.js?v=20241107.3">
</script>
```

```csharp
bundles.CreateJsBundle("main",
    "~/js/app.js",
    "~/js/vendor.js",
    "~/js/jquery.js");
```

::right::

<v-click>

```
dist/assets/
  main-a3f9c21b.js
  main-7d02e114.css
```

```js[vite.config.js]
build: {
  manifest: true,
  rollupOptions: { input: 'src/main.js' }
}
```

```html[layout.cshtml]
<script type="text/javascript" asp-src-include="/assets/js/main-*.js" defer></script>
```

</v-click>

<!--
Bundling is one of my pet peeves. **Server-side bundling, with cache busting by build number**, was the best way to do things for a long, long time. **Smidge was removed from Umbraco in v14**, when the backoffice moved to ES modules that server-side bundling can't handle, and Umbraco now recommends a front-end build tool instead. There's a community package to put it back... but I wouldn't.

It doesn't do tree shaking. **The cache busting is global: one version number, so every asset invalidates on every deploy** - cachebusting per deploy, when it should be per file. **It needs .NET to understand JavaScript and CSS**, so when new front-end features come along, it breaks. And if you bring a modern front-end developer onto your team, this isn't how they expect bundling to work.

[Click] Here I'm using Vite. It doesn't have to be Vite - it's just what we use, and what Umbraco's own docs recommend for backoffice extensions. **The file names are content-hashed, so they only change when the file changes.** It tree-shakes, so code you've stopped using leaves the bundle. And it's really easy to reference from .NET: **the script tag helper's asp-src-include takes a glob**, so main-*.js picks up whatever the current file is.
-->

---
layout: two-cols-header
---

# jQuery
<small>The ghost of `$(document).ready` (2016, if we're honest)</small>

::left::

```js
$(document).ready(function () {
  $('.nav-toggle').on('click', function () {
    $('.nav').toggleClass('open');
  });
});
```

::right::

<v-click>

```js
document.querySelector('.nav-toggle')
  .addEventListener('click', () =>
    document.querySelector('.nav')
      .classList.toggle('open'));
```

</v-click>

<!--
jQuery. It's still in a lot of the long-lived sites we work on. **jQuery was built largely to paper over the differences between browsers - Internet Explorer above all.** Internet Explorer is dead, and the remaining browser engines are all very standards-compliant.

And **we don't need document ready** if we load our scripts with defer, or as modules - the document's been parsed by the time they run.

[Click] **Same length, one fewer dependency.** Selectors, event binding, toggling a class - are all one-liners in modern JavaScript now. And **modern JavaScript works alongside jQuery, so new code doesn't have to be jQuery. You can replace it gradually.**

Did you know a new version of jQuery came out this year? **jQuery 4 came out in January - the first major version in almost a decade.** It drops old Internet Explorer and a lot of deprecated APIs, so it's **lighter**. If you can't get rid of jQuery yet, at least upgrade it. There may be some compatibility issues with plugins, but it's almost free. **Preferably, get rid of it, step by step.**
-->

---
layout: two-cols-header
---

# Rolling your own
<small>The ghost of "it's only a few classes" (2025, the model wrote it)</small>

::left::

> "It's only a few classes"

<br />

> "The package does more than we need"

<br />

> "I don't want another dependency"

<br />

> "…the model wrote it in a minute"

::right::

<v-click>

| Signal | Good | Bad |
| --- | --- | --- |
| Downloads | Growing | Flat since v8 |
| Last commit | Weeks | Years |
| Maintainers | More than one | Exactly one |
| Issues | Answered in days | Silence |
| Latest version support | Supported | Not mentioned |

</v-click>

<!--
Everyone has said or thought at least two of these. **Rolling your own is so tempting**: it's only a few classes, and now the LLM will write it in a few minutes. **The argument for building it yourself just got a lot more persuasive, because the cost of building collapsed.**

But **you didn't save the week. You bought a decade of maintenance.** The value of a package was never the code - **it's the bug reports from everyone else's sites.** Hundreds of sites have already hit the edge cases you haven't reached yet, and you can't generate that. When the next major version lands, either the maintainer does the work or you do. **Generated code is cheap to write and exactly as expensive to own.** And if you have built something with AI, **open source it**, so we all share the maintenance, the testing and the new features.

[Click] So keep looking for packages, and here's how to judge one. Downloads active and growing. A recent last commit. **Preferably more than one maintainer** - though some single maintainers' reputations do precede them. Issues answered, not ignored. And even if you're not on the latest Umbraco yet, **make sure it supports the latest version, so you know it'll be there when you upgrade.**
-->

---
layout: intro
# ghostly fade between sections

---

# This *will* be wrong by 2027

<!--
And it's very important to say: **this talk will be wrong by next year.**

That's fine. **The ground hasn't stopped moving - that's what a platform that's still alive looks like.** **The failure isn't having written the old pattern. It's never going back to look.**
-->

---
layout: intro
---

# The Ghosts of Umbraco Yet to Come

<!--
So now let's move onto the Ghosts of Umbraco Yet to Come - how do we spot the next one, without waiting for me to rehash this talk in a year or two?
-->

---

# The Ghosts of Umbraco Yet to Come

<v-clicks>

1. **Is the behavior hiding something?**
2. **How old is the advice?** Date the blog post, the SO answer, the colleague
3. **What changed underneath?** Cache, runtime, language, backoffice, hosting
4. **Who is this convenient for?** Developer convenient, editor expensive?
5. **Am I just being nostalgic?** If everyone else is doing something different and you're longing for "the old way", is it just nostalgia?

</v-clicks>

<!--
Here's a checklist that I believe may help spot ghosts. [Click] **Is the behaviour hiding something?** If something's tucked away in a service, or the return type isn't telling you what you're actually getting - like that IEnumerable that was secretly a list - that's a red flag.

[Click] **How old is the advice?** How old is the blog post, the Stack Overflow answer, the forum post? I did my first training course on v7 - a lot of that is out of date now! And how recently has your colleague taken in anything from the community? You're all getting fresh advice today.

[Click] **What changed underneath?** Keep a vague idea of the big headline changes in each version, in the back of your mind. The cache changed - so is this still the right way to use the cache?

[Click] **Who is this convenient for?** If it's convenient for the developer, great - but is it at the expense of the editor?

[Click] And this is an interesting one: **am I just being nostalgic?** If the whole community has moved on and you're still doing it the old way, **is there a reason beyond "that's how I've always done it"?** And the AI assistants confidently recommending v7 patterns are making exactly the same mistake, automated - trained on a decade of blog posts that were right when they were written.
-->

---
layout: bio
name: Thank you!
image: /images/joe-portrait.png
---

- <logos-mastodon-icon /> [@joe@umbraco&#8239;community.social](https://umbracocommunity.social/@joe)
- <logos-bluesky /> [@joe.gl](https://bsky.app/profile/joe.gl)
- <solar-global-bold-duotone /> [joe.gl/ombek](https://joe.gl/ombek)

<br />

- <solar-presentation-graph-bold-duotone/> https://slides.joe.gl/ghosts-of-umbraco-past

<!--
**Thank you very much for listening.** There's no Q&A, so **come and find me in the hallway, or on socials.**
-->
