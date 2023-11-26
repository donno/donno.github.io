---
layout: post
title:  "Jekyll Update"
date:   2023-11-25 26:40:00 +0930
---

This weekend when I went to deploy my latest blog post, it didn't work. I
pushed, refreshed and there was no new post on the site. Assuming that the
GitHub Action was sitting in a queue, I went to bed with the idea of checking
on it in the morning. When I checked it still wasn't there.

## Deployment Issue
I checked the [Actions in GitHub][1], it reported the build and deploy was
successful. Checking the "Build with Jekyll" stage of the Build job, I noticed
in the file list of HTML pages, that the new post was missing.
Scrolling up further, there was this line:
> Warning:  github-pages can't satisfy your Gemfile's dependencies.

I checked the issues against `actions/jekyll-build-pages` which is the GitHub
action used for deploying my site and there was [#104][7] which mentioned the
warning and how to improve it, but not what the probable cause was.

I hadn't changed any of the dependencies when making my last post so that was
odd nor did I seem to be using any non-standard dependencies. Building locally
pointed out the issue was it was missing `webrick`, a HTTP server toolkit for
Ruby. This [StackOverflow post][2] and [GitHub issue][3] mentions that the fix
is to add `webrick` and it is caused by the use of Ruby 3.0. This itself is
strange as jekyll-build-pages:v1.0.9 is suppose to use Ruby 2.7.4.
The image my action refers is ubuntu-latest which was is now ubuntu-22.04 and
based on the [software list][4], it contains Ruby 3.0.2p107 with 3.1.4 also
available.

To have the post published, the dependency of `webrick` was added and the new
post went live.

When preparing this post, I came across the [GitHub Pages documentation][5]
calls out this issue and the fix and so does the [Jekyll][6] documentation.
It means with Ruby 3.0 or later you may get this error as it no longer comes
with `webrick` installed and you can fix it by running `bundle add webrick`

## Action Upgrades
It has been over a year since I set-up the blog and actions so it was overdue
for updates. I started out by updating the  actions, this was the a straight
forward and an easy change to make and test.

* `actions/upload-pages-artifact` from v1 to v2
* `actions/deploy-pages` from v1 to v2
* `actions/checkout from` v3 to v4
* `actions/configure-pages` from v2 to v3

The key change for `checkout` v3 to v4 was switching from Node.js 16.x to 20.x,
as Node 16 reached end-of-life on 11 Sep 2023. The next LTS was 18 however
action provider skipped that version and went to 20 which is still in active
support.

The main change `upload-pages-artifact` v2 brings over v1 is the removal of
the `chmod` command built into the action. According to the changelog this
was slowing down deployment, which I assume was mainly large user, and it
was also causing issues where it was setting the permission incorrectly on
certain files. The recommendation going forward is to manually set the
permission yourself when needed.

The chmod issue was raising various messages about in the form of:
`chmod: changing permissions of '<path>': Operation not permitted`
After updating the dozen or so lines disappeared from the log and there was no
obvious downside.

## Jekyll Upgrade

Jekyll itself is where things get harder. It was not a matter of changing
`actions/jekyll-build-pages` from v1 to v2 because they have not produced a v2
that goes from jekyll v3.x to v4.x. This is because the action depends on
[github-pages][8] and based on [issue #651][9], there is no plans to make
a new version that uses Jekyll 4. The solution is to switch from using
`github-pages` which from what I can tell is mostly a package that depends on
several themes, plugs and so on. The gem is meant as a quick and easy way of
getting a reasonably nice Jekyll setup out-of-the-box with several themes to
make it easy to switch themes. I suspect this was for the pre-GitHub action set
up of GitHub pages, so it tried to come with a reasonable set of themes and
plugins.

### Changes
Gemfile changes
* Reference `jekyll 4.3.0`
* Remove `github-pages` - this package depends on `jekyll`.
* Update `minima` - this is the theme used on the blog.

Action changes
* Remove `actions/jekyll-build-pages@v1`
* Use [`ruby/setup-ruby@v1`][10] - this is required to install bundle
* Add new build step that runs `bundle exec jekyll build`

The other change made to get this to work was to move the Gemfile from the docs
subdirectory to the top-level of the repository. This was needed for the
`ruby/setup-ruby` action as it runs from there instead of the `docs`
subdirectory and it moving the file was simple and sticks to the common
location where it expects and where it would be for others. The action supports
 `working-directory` as an input so it wouldn't take much to customise it.

After a couple of deploy failures which discovered the `bundle` command was
missing and the Gemfile was not where the command expected. This should put
it far enough ahead now that it does not need to updating again for a while.

[1]: https://github.com/donno/donno.github.io/actions
[2]: https://stackoverflow.com/a/66013726
[3]: https://github.com/jekyll/jekyll/issues/8523
[4]: https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md
[5]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll
[6]: https://jekyllrb.com/docs/
[7]: https://github.com/actions/jekyll-build-pages/issues/104
[8]: https://github.com/github/pages-gem
[9]: https://github.com/github/pages-gem/issues/651
[10]: https://github.com/ruby/setup-ruby/