[![Build Status](https://travis-ci.org/pages-themes/architect.svg?branch=master)](https://travis-ci.org/pages-themes/architect) [![Gem Version](https://badge.fury.io/rb/jekyll-theme-architect.svg)](https://badge.fury.io/rb/jekyll-theme-architect)

[![Thumbnail of architect](thumbnail.png)](http://pages-themes.github.io/architect)

* [Introduction](#introduction)
* [Quick start](#quick-start)
* [Step by step](#step-by-step)


# Introduction
*The repository is used to record my study, it is a static [blog][] site based on the [jekyll-theme-architect][], if you don't want to waste much time on create blog step by step, you can  [use it today](#quick-start).*

[blog]: https://donaldhan.github.io/ "Donald Blog"
[jekyll-theme-architect]: http://pages-themes.github.io/architect "jekyll-theme-architect"
## Quick start

To create blog base the repository :
1. you can pull the repository, and modify repository name to yours, such as:
    Donaldhan.github.io -> jamel.github.io
2. modify some site constants in `_config.yml` under the root, `profile.yml` and `gitalk.yml` under the \_data dir.
the `_config.yml` the global config of site, the laters are  private. Being care for the `gitalk.yml`, it is a comment  plugin of article base the [gitalk][].
3. install Jekyll, [Runnig Jekyll on Linux][linux-jekyll],[Running Jekyll on Windows][windows-jekyll].
4. Clone down your repository
5. `cd` into your local repository directory
6. Run `bundle exec jekyll serve` to start the preview server
5. Visit [`localhost:4000`](http://localhost:4000) in your browser to preview your blog.

[gitalk]: https://github.com/gitalk/gitalk
[linux-jekyll]: https://jekyllrb.com/docs/installation/ "Runnig Jekyll on Linux"
[windows-jekyll]: http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html "Running Jekyll on Windows"

## Step by step
If you want to create blog by yourself from zero, or you are stirring guy, you can reference the [notes][notes_url] on my gitee repository.   
Before you start to create blog, you must realize that what jekyll is? Simplely ,it is a blog tool drives by ruby, use your blog repository to produce a static site. Then [yaml][] and [markdown][], `yaml`  is a human friendly data serialization standard for all programming languages, likes `xml`, but simplely, describle properties use some special symbols , instead of 'xml' use the complicate label, and you have to anlysize. `markdown` is  an easy-to-read and easy-to-write plain text marked language, and it is feasible, compatible with html. Of course, you need learn [Liuqid][]. If you want to edit markdown file, some software can help you , such as [Atom][] , [MarkdownPad][], `atom` is A github offical hackable text editor for markdown, has linux and windows two version. `markdownPas` is for windows, suggest to atom.   
After above, you can create a simple static blog site, if you want some especial function ,such as fulltext search ([Jekyll Tipue Search][tipue-search]) and article comment([gitalk][]).

[notes_url]: https://gitee.com/Donaldhans/draft/blob/master/git-page-blog.md
[yaml]: http://www.yaml.org/ "YAML"
[markdown]: https://daringfireball.net/projects/markdown/syntax "Markdown"
[Liuqid]: https://help.shopify.com/themes/liquid/basics "Liuqid"
[tipue-search]: https://github.com/jekylltools/jekyll-tipue-search "Jekyll Tipue Search based liquid"
