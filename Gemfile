source "https://rubygems.org"

gem "jekyll", "~> 3.10"
#gem "minimal-mistakes-jekyll"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
  gem 'github-pages', "~> 232"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end
