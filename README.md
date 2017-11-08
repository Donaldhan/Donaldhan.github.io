# Introduction

[![Build Status](https://travis-ci.org/pages-themes/architect.svg?branch=master)](https://travis-ci.org/pages-themes/architect) [![Gem Version](https://badge.fury.io/rb/jekyll-theme-architect.svg)](https://badge.fury.io/rb/jekyll-theme-architect)

[![Thumbnail of architect](thumbnail.png)](http://pages-themes.github.io/architect)

*The repository is used to record my study， it is a static site based on the [jekyll-theme-architect](http://pages-themes.github.io/architect).,
if you don't want to waste much time on create blog step by step, you can  [use it today](#quick-start).*

## Quick Start

To create blog base the repository :
1. you can pull the repository, and modify repository name to yours, such as:
    Donaldhan.github.io -> jamel.github.io
2. modify some site constants in `_config.yml` under the root, `profile.yml` and `gitalk.yml` under the \_data dir.
the `_config.yml` the global config of site, the laters are  private. Being care for the `gitalk.yml`, it is a comment  plugin of article base the [gitalk](https://github.com/gitalk/gitalk).
3. install Jekyll, [Runnig Jekyll on Linux](https://jekyllrb.com/docs/installation/),
[Running Jekyll on Windows](http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html).
4. Clone down your repository
5. `cd` into your local repository directory
6. Run `bundle exec jekyll serve` to start the preview server
5. Visit [`localhost:4000`](http://localhost:4000) in your browser to preview your blog.

## Step by Step
if you want to create blog by yourself step by step, or you are stirring guy, you can reference the [notes]
(https://gitee.com/Donaldhans/draft/blob/master/git-page-blog.md)
on my gitee repository.

1. Add the following to your site's `_config.yml`:

    ```yml
    theme: jekyll-theme-architect
    ```

2. Optionally, if you'd like to preview your site on your computer, add the following to your site's `Gemfile`:

    ```ruby
    gem "github-pages", group: :jekyll_plugins
    ```



## Customizing

### Configuration variables

Architect will respect the following variables, if set in your site's `_config.yml`:

```yml
title: [The title of your site]
description: [A short description of your site's purpose]
```

Additionally, you may choose to set the following optional variables:

```yml
show_downloads: ["true" or "false" to indicate whether to provide a download URL]
google_analytics: [Your Google Analytics tracking ID]
```

### Stylesheet

If you'd like to add your own custom styles:

1. Create a file called `/assets/css/style.scss` in your site
2. Add the following content to the top of the file, exactly as shown:
    ```scss
    ---
    ---

    @import "{{ site.theme }}";
    ```
3. Add any custom CSS (or Sass, including imports) you'd like immediately after the `@import` line

### Layouts

If you'd like to change the theme's HTML layout:

1. [Copy the original template](https://github.com/pages-themes/architect/blob/master/_layouts/default.html) from the theme's repository<br />(*Pro-tip: click "raw" to make copying easier*)
2. Create a file called `/_layouts/default.html` in your site
3. Paste the default layout content copied in the first step
4. Customize the layout as you'd like

## Roadmap

See the [open issues](https://github.com/pages-themes/architect/issues) for a list of proposed features (and known issues).

## Project philosophy

The Architect theme is intended to make it quick and easy for GitHub Pages users to create their first (or 100th) website. The theme should meet the vast majority of users' needs out of the box, erring on the side of simplicity rather than flexibility, and provide users the opportunity to opt-in to additional complexity if they have specific needs or wish to further customize their experience (such as adding custom CSS or modifying the default layout). It should also look great, but that goes without saying.

## Contributing

Interested in contributing to Architect? We'd love your help. Architect is an open source project, built one contribution at a time by users like you. See [the CONTRIBUTING file](CONTRIBUTING.md) for instructions on how to contribute.

### Previewing the theme locally

If you'd like to preview the theme locally (for example, in the process of proposing a change):

1. Clone down the theme's repository (`git clone https://github.com/pages-themes/architect`)
2. `cd` into the theme's directory
3. Run `script/bootstrap` to install the necessary dependencies
4. Run `bundle exec jekyll serve` to start the preview server
5. Visit [`localhost:4000`](http://localhost:4000) in your browser to preview the theme

### Running tests

The theme contains a minimal test suite, to ensure a site with the theme would build successfully. To run the tests, simply run `script/cibuild`. You'll need to run `script/bootstrap` one before the test script will work.
