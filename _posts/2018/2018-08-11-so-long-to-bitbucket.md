---
layout: post
title: So long to Bitbucket
tags:
- blog
- jekyll
- bitbucket
- github
---

## The Decision

We've been using bitbucket.org for the last year and a half.  The decision was partially driven by our use of other Atlassian products (notably Jira and Confluence) including [Bitbucket Server](https://www.atlassian.com/software/bitbucket/server) (formerly "Stash"), but also spurred by its use by two of our partners.

While I think everyone here has adjusted to git and is pretty happy with it, bitbucket itself leaves a lot to be desired.  My gripes boil down to (in no particular order):
- Unexpected rendering of markdown (at least, it differs from [VS Code](https://code.visualstudio.com/Docs/languages/markdown))
- Mediocre issues, releases, and general community functionality
- Clumsy static site hosting

Regardless of how you feel emotionally about [the acquisition](https://news.microsoft.com/2018/06/04/microsoft-to-acquire-github-for-7-5-billion/), it's tough to deny that github is decidely better than bitbucket.

Our public announcement meant that we could move from private to public repositories and it seemed like the opportune time to [move to github](https://github.com/subor).

For the sake of continuity it makes sense to do the same with this blog.

## Moving Day

I've got a heaping backlog of notes to turn into posts, but I haven't been good on follow-through.

Using [Jekyll's notes on upgrading](https://jekyllrb.com/docs/upgrading/), I tried to update my MBP with both `bundle update jekyll` and `bundle update` but was rebuffed with `Could not locate Gemfile`.

Not entirely surprised there's something wrong with my Ruby environment since I never use the language.

`gem update jekyll`:

	ERROR:  While executing gem ... (Gem::FilePermissionError)
	    You don't have write permissions for the /Library/Ruby/Gems/2.3.0 directory.

Nope.  `sudo gem update jekyll`:

	ERROR:  While executing gem ... (Gem::FilePermissionError)
	    You don't have write permissions for the /usr/bin directory.

Nope.  `sudo gem update jekyll -n /usr/local/bin`:

Bingo.

Unlike with bitbucket there's a bevy of resources about hosting static websites on Github:
- [https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)
- [http://jmcglone.com/guides/github-pages/](http://jmcglone.com/guides/github-pages/)
- [https://jekyllrb.com/docs/github-pages/](https://jekyllrb.com/docs/github-pages/)

The best part being Github takes care of running Jekyll for you.  So, rather than needing to juggle markdown, generation, and html with bitbucket, I can just push the markdown to Github.

### _config.yml

When I first pushed the site, nothing was displayed.  After poking around I noticed `url: "https://rendered-obsolete.bitbucket.io"`, changed it to `url: "https://rendered-obsolete.github.io"`, and pushed the site again.  Now it displayed, however I'm not really sure if that was necessary or not.

And with that I bring you [https://rendered-obsolete.github.io/](https://rendered-obsolete.github.io/)!
