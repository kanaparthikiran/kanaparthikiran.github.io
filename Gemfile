source "https://rubygems.org"

# GitHub Pages tracks a specific gem set; using github-pages keeps local
# preview aligned with what GH Pages actually builds.
gem "github-pages", group: :jekyll_plugins

# Optional: tweaks for tooling on macOS
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Performance fix for Jekyll on M-series Macs
gem "webrick", "~> 1.8"

# Standard-library modules that Ruby 3.4+ moved out of default gems but
# Jekyll 3.9 (pinned by github-pages) still expects.
gem "csv"
gem "base64"
gem "bigdecimal"
gem "logger"
gem "mutex_m"
gem "ostruct"

# Lock to Ruby that GitHub Pages actually uses server-side
ruby ">= 3.1"
