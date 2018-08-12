---
layout: post
title: GitHub Pages Migration Part 2
tags:
- github
- blog
- jekyll
---

This is a follow-up to my [previous post about migrating to Github Pages]({% post_url /2018/2018-08-11-so-long-to-bitbucket %})

## On Bundle and Gemfiles

As is so often the case, I figured out my problem with `bundle` a few hours later.  While looking into something else I came across [The github-pages gem](https://jekyllrb.com/docs/github-pages/#the-github-pages-gem)- which I evidentally missed the first time I set this up- and was reminded that the `Gemfile` was in my own repo.  When executed in the right place `sudo bundle update` works as expected.

Working with new tech is always a humbling experience.

## On Repositories

Anyway, upgrade to Jekyll 3.7.3 and...

```
  Liquid Exception: No repo name found. Specify using PAGES_REPO_NWO environment variables, 'repository' in your configuration, or set up an 'origin' git remote pointing to your github.com repository. in /_layouts/default.html

ERROR: YOUR SITE COULD NOT BE BUILT:
       ------------------------------------
       No repo name found. Specify using PAGES_REPO_NWO environment variables, 'repository' in your configuration, or set up an 'origin' git remote pointing to your github.com repository.
```

[This Jekyll issue](https://github.com/jekyll/jekyll/issues/4705) and the [PR it referenced](https://github.com/jekyll/github-metadata/pull/45) clarifies what is going on.

To move to github I'd added another remote git repository `github` (my previous bitbucket repo was `origin`).  Like the helpful ERROR message states, I either need to set __repository__ in `_config.yml` or the git remote named `origin` needs to point to github.com.

Simple enough.

## Theming

Now looking at [a GitHub doc on Jekyll theming](https://help.github.com/articles/adding-a-jekyll-theme-to-your-github-pages-site/).

In `_config.yml` switching `theme: minima` for `theme: jekyll-theme-midnight`.

```
Build Warning: Layout 'post' requested in _drafts/2018/2018-08-12-github-pages-migration.md does not exist.
Build Warning: Layout 'page' requested in about.md does not exist.
Build Warning: Layout 'home' requested in index.md does not exist.
```
And a blank site.

The line about `index.md` is straight-foward.  I removed `layout: home` from the [YAML front matter](https://jekyllrb.com/docs/frontmatter/) and now had an empty index page.

There's [several issues](https://github.com/benbalter/wordpress-to-jekyll-exporter/issues/37) about non-existant layouts and from the [Jekyll doc of overriding themes](https://jekyllrb.com/docs/themes/#overriding-theme-defaults) I find my __minima__ theme has:
```
$ ls `bundle show minima`/_layouts
default.html	home.html	page.html	post.html
```
While __midnight__ has:
```
$ ls `bundle show jekyll-theme-midnight`/_layouts 
default.html
```

Fine:
```
cp `bundle show minima`/_layouts/home.html _layouts/
cp `bundle show minima`/_layouts/post.html _layouts/
cp `bundle show minima`/_layouts/page.html _layouts/
```

And everything is working.

## Pagination

Having one long index with every post is starting to look a bit silly.  There's the [Jekyll doc on pagination](https://jekyllrb.com/docs/pagination/).  It recommends jekyll-paginate which states that it is deprecated in favor of [jekyll-paginate-v2](https://github.com/sverrirs/jekyll-paginate-v2) (which is [not yet supported by GitHub Pages](https://pages.github.com/versions/)).

In `_config.yml`:
```
plugins:
  - jekyll-paginate
paginate: 6
```
`jekyll serve` complains:
```
Pagination: Pagination is enabled, but I couldn't find an index.html page to use as the pagination template. Skipping pagination.
```

Since my `index.md` just contains `layout: home`, I move `_layouts\home.html` to `index.html` (in the root of the repo).  And adding the pagination templating they recommend.

Works, but now all the posts are paginated inline on the index page.  Not sure that's what I want so I'm backing that out.  I'm trying to avoid fiddling with visual layout as much as possible, but I'm finding it harder and harder to retain this policy.