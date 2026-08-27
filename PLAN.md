Keep it technical, all patterns we’ve seen and done but theres better ways now.

Common Pattern, Why this is wrong (performance, maintainability - incl. AI, cost), How do i fix it?



- Limiting your LINQ https://trello.com/c/yACQylAq/16-limiting-your-linq - UmbracoFriendlyExtensions cacheing changes
- DateTimeOffset over DateTime https://trello.com/c/nAfjr77b/1-datetime-is-so-datetimenowutcaddyears-1
- .ToList/.ToArray everywhere
- .Union or .AppendMany → [.. collectionA, ..collectionB, singleItem] spread operator
- collection expression (IReadOnlyDictionary soon)
- Repeated ContentFinders, 404s (Virgin /nyc on a billboard), DB calls for Redirects (Skybrud has no cache)
  - Delivery API for redirects → static site route table (/CloudFlare Worker?)
- Smidge
  - Modern FE bundling (tree shaking!); hash versioning, not build numbers
- Back office load balancing
- Building your own packages (AI)
  - how to pick a package, Stars, downloads, latest commits, maintainers
- ViewComponents over Partials etc
- Doc Type inheritance
- Poorly named:
  - document types & compositions - confusing interface names, clashing names - conventions “X Page”, “X Item” - nouns not implementation details
  - Blog, Blog Post/Article reads nicer than “Blog Listing”
  - Stick to nouns, not the implementation details?
  - - Using different alias to names
    - Paul Seal’s blog post Callum's rant
- Override block names in grid/list to make it clear to editors/contextual
  - Don't have to use different alias to name
- Block labels
  - use them
  - sensible fallbacks - almost never the name
- Global stylesheets
  - apply targeted styles in context
  - block grid blocks can have individual stylesheets
  - RTE - approximation of frontend, not all your styles
- WYSIWYG
  - content modelling and reuse
  - data architecture
  - make the data structure make sense, the page will build itself
- jQuery
  - lay to rest in 2016!
  - upgrade to v latest