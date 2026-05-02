source "https://rubygems.org"

# `github-pages` is a meta-gem that pins Jekyll, Minima, and every whitelisted
# plugin to the exact versions GitHub Pages runs server-side. This keeps your
# local build identical to production. See https://pages.github.com/versions/
gem "github-pages", group: :jekyll_plugins

# webrick was removed from Ruby stdlib in 3.0 but Jekyll 3 (the version github-pages
# pins to) still uses it for `jekyll serve`. Without this, local preview crashes.
gem "webrick", "~> 1.8"

# Plugins not bundled with github-pages can be added below in the same group.
# group :jekyll_plugins do
#   gem "jekyll-paginate-v2"
# end
