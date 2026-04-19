About
=====

The website for GitHub user 'donno'.

This website is built with Jekyll 4+.

Notes
-----
- A GitHub Workflow is used to build and deploy the site to GitHub Pages.
- Testing locally before deployment: `bundle exec jekyll serve` from the
  working tree root directory.


Building
--------
```sh
podman run --rm --volume=$(pwd):/srv/jekyll -it docker.io/bretfisher/jekyll:latest jekyll build
```

Why not `jekyll/jekyll`? It has essentially been abandoned for over 3 years
and the maintainer stepped away. It has Ruby 3.1 which went end-of-life in
April 2025.

After updating the versions in the `Gemfile`, run `bundle update --all`.
