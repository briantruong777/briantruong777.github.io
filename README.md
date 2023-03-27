Brian's Personal Website
========================

This is the repository for my personal website: [briantruong777.github.io].
It's a standard [GitHub Pages] repository using [Jekyll].

[briantruong777.github.io]: https://briantruong777.github.io
[GitHub Pages]: https://pages.github.com/
[Jekyll]: https://jekyllrb.com/

# Installation

These instructions are paraphrased from the [official documentation].

1. Install [Ruby].
1. Install [bundler].
1. Run `bundle install` in the root of the repository.

[official documentation]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll
[Ruby]: https://www.ruby-lang.org/
[bundler]: https://bundler.io/

Note that you will need the [specific Ruby version that GitHub Pages uses](https://pages.github.com/versions/)
which may require something like [rbenv] and [ruby-build] if your system
doesn't support that version.

[rbenv]: https://github.com/rbenv/rbenv
[ruby-build]: https://github.com/rbenv/ruby-build

# Run Locally

Serve the website locally with change watching, live reload, and include draft
posts.

```
$ bundle exec jekyll serve --watch --livereload --draft
```

Update all dependencies to their latest version.

```
$ bundle update
```

# License

Unless otherwise noted, the contents of my webpage are licensed under the
[Creative Commons Attribution 4.0 International License]. Code used to format
and display the content is licensed under the [MIT license].

[Creative Commons Attribution 4.0 International License]: https://creativecommons.org/licenses/by/4.0/
[MIT license]: https://opensource.org/licenses/MIT
