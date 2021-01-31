source "https://rubygems.org"

gem "jekyll", "~> 4.1"

gem "jekyll-theme-chirpy"

# plugins
group :jekyll_plugins do
  gem "jekyll-paginate", "~> 1.1"
  gem "jekyll-redirect-from", "~> 0.16"
  gem "jekyll-seo-tag", "~> 2.7"
  gem "jekyll-archives", "~> 2.2"
  gem "jekyll-sitemap", "~> 1.4"
end

install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end
gem "wdm", "~> 0.1.0", :install_if => Gem.win_platform?
gem "kramdown-parser-gfm"

