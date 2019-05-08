
# Hydejack

https://github.com/qwtel/hydejack/releases

# Hydejack Starter Kit

https://github.com/qwtel/hydejack-starter-kit

# files

    $ mkdir _nojekyll && mv -iv docs.md about.md index.html _featured_categories/example.md assets/img/ example/ _nojekyll/
    ‘docs.md’ -> ‘_nojekyll/docs.md’
    ‘about.md’ -> ‘_nojekyll/about.md’
    ‘index.html’ -> ‘_nojekyll/index.html’
    ‘_featured_categories/example.md’ -> ‘_nojekyll/example.md’
    ‘assets/img/’ -> ‘_nojekyll/img’
    ‘example/’ -> ‘_nojekyll/example’

    $ git status
    # On branch master
    # Changes not staged for commit:
    #   (use "git add/rm <file>..." to update what will be committed)
    #   (use "git checkout -- <file>..." to discard changes in working directory)
    #
    #       modified:   .gitignore
    #       modified:   README.md
    #       modified:   _config.yml
    #       modified:   _data/authors.yml
    #       modified:   _data/strings.yml
    #       deleted:    _featured_categories/example.md
    #       modified:   _sass/my-inline.scss
    #       deleted:    about.md
    #       deleted:    assets/img/hydejack-8.jpg
    #       deleted:    assets/img/hydejack-8@0,25x.jpg
    #       deleted:    assets/img/hydejack-8@0,5x.jpg
    #       deleted:    docs.md
    #       deleted:    example/_posts/2017-11-23-example-content-ii.md
    #       deleted:    example/_posts/2018-06-01-example-content-iii.md
    #       modified:   index.html
    #
    # Untracked files:
    #   (use "git add <file>..." to include in what will be committed)
    #
    #       _featured_categories/category.md
    #       _featured_categories/tag.md
    #       _posts/
    #       aboutme.md
    #       assets/icons/
    #       atom.xml
    no changes added to commit (use "git add" and/or "git commit -a")

    $ tail .gitignore

    # delete from upstream example files
    _featured_categories/example.md
    _nojekyll/*
    assets/img/*
    example/*
    about.md
    docs.md

    $ find .|egrep -v '.git|_nojekyll'
    .
    ./_config.yml
    ./_includes
    ./_includes/my-body.html
    ./_includes/my-head.html
    ./_plugins
    ./_plugins/jekyll-replace-imgs
    ./_plugins/jekyll-replace-imgs/jekyll-replace-imgs.rb
    ./_plugins/README.md
    ./atom.xml
    ./Gemfile.lock
    ./index.html
    ./_featured_categories
    ./_featured_categories/category.md
    ./_featured_categories/tag.md
    ./_sass
    ./_sass/my-style.scss
    ./_sass/my-inline.scss
    ./aboutme.md
    ./assets
    ./assets/icons
    ./assets/icons/favicon.ico
    ./Gemfile
    ./_posts
    ./_posts/2018-08-31-openssh-7.8p1-broken-pipe-under-vmware-vm-with-nat-port-forward.md
    ./_data
    ./_data/social.yml
    ./_data/authors.yml
    ./_data/strings.yml
    ./README.md
    ./404.md

# update

https://help.github.com/en/articles/syncing-a-fork

    git remote add upstream https://github.com/qwtel/hydejack-starter-kit.git
    git fetch upstream
    git status
    git merge upstream/master

    git log --graph --date=short -6

[Fork 的分支从源分支更新的方法](https://github.com/BearRan/CRAnimation/wiki/Fork的分支从源分支更新的方法)

[gitlab 或 github 下 fork 后如何同步源的新更新内容？](https://www.zhihu.com/question/28676261/answer/44606041)

# rebuild

download old repo :

    mkdir rebuild && cd rebuild
    wget https://github.com/lvii/lvii.github.io/archive/master.zip && unzip -x master.zip

refork origin repo :

    cd ~ && git clone git@github.com:lvii/lvii.github.io.git
    rsync -avP -n lvii.github.io-master/ ~/lvii.github.io

# jekyll

## var

https://jekyllrb.com/docs/variables/

https://stackoverflow.com/questions/6366188/jekyll-select-current-page-url-and-change-its-class

    {{ site.url }}
    {{ page.url | remove:'index.html' }}
    {{ page.relative_path }}
    {{ page.relative_path | replace:'index.html', '' | remove: '_blocks/' | prepend: site.baseurl }}

## font

`_config.yml` 引用 google 宋体：

    google_fonts:          Noto+Serif+SC:400,700|PT+Sans:400,400i,700|Roboto+Slab:700|Noto+Sans:400,400i,700,700i

修改 `_sass/my-inline.scss` 标题列表对应的 CSS 样式：

    .title-list li a {
      display: block;
      font-weight: 700;
      line-height: 1.75;
      padding: .25rem 0;
      font-family: Roboto Slab, Noto Serif SC, Helvetica, Arial, sans-serif !important;
    }

https://fonts.google.com/specimen/Noto+Serif+SC

    <link href="https://fonts.googleapis.com/css?family=Noto+Serif+SC:400,700" rel="stylesheet">

    font-family: 'Noto Serif SC', serif;

![img](https://i.imgur.com/5TguCJU.png)

[Google Fonts 已支持思源宋体！2018-12-11](https://reuixiy.github.io/beautiful/share/2018/12/11/noto-serif-sc-added-on-google-fonts.html)

## autolink

    $ git diff _config.yml

    +
    +# URLs are not autolinked in GFM mode #306
    +# https://github.com/gettalong/kramdown/issues/306
    +markdown: CommonMarkGhPages
    +commonmark:
    +   extensions:
    +   - autolink

    $ git diff Gemfile
    diff --git a/Gemfile b/Gemfile
    index 91e0a43..d7c1d83 100644
    --- a/Gemfile
    +++ b/Gemfile
    @@ -29,6 +29,7 @@ group :jekyll_plugins do
       gem "jekyll-seo-tag"
       gem "jekyll-sitemap"
       gem "jekyll-titles-from-headings"
    +  gem "jekyll-commonmark-ghpages"
     end

## install

    $ bundle install
    Warning: the running version of Bundler (1.13.7) is older than the version that created the lockfile (1.16.1).
    We suggest you upgrade to the latest version of Bundler by running `gem install bundler`.
    Fetching gem metadata from https://rubygems.org/.............
    Fetching version metadata from https://rubygems.org/...
    Fetching dependency metadata from https://rubygems.org/..
    Resolving dependencies...
    Using public_suffix 3.0.2
    Using colorator 1.1.0
    Using concurrent-ruby 1.0.5
    Using eventmachine 1.2.7
    Using http_parser.rb 0.6.0
    Using multipart-post 2.0.0
    Using ffi 1.9.25
    Using forwardable-extended 2.6.0
    Using rb-fsevent 0.10.3
    Using ruby_dep 1.5.0
    Using kramdown 1.17.0
    Using liquid 4.0.0
    Using mercenary 0.3.6
    Using rouge 3.1.1
    Using safe_yaml 1.0.4
    Using jekyll-paginate 1.1.0
    Using rubyzip 1.2.1
    Using bundler 1.13.7
    Using addressable 2.5.2
    Using i18n 0.9.5
    Using em-websocket 0.5.1
    Using faraday 0.15.2
    Using rb-inotify 0.9.10
    Using pathutil 0.16.1
    Installing ruby-enum 0.7.2
    Using sawyer 0.8.1
    Using sass-listen 4.0.0
    Using listen 3.1.5
    Installing commonmarker 0.17.13 with native extensions
    Using octokit 4.9.0
    Using sass 3.5.6
    Using jekyll-watch 2.0.0
    Using jekyll-gist 1.5.0
    Using jekyll-sass-converter 1.5.2
    Using jekyll 3.8.3
    Using jekyll-avatar 0.6.0
    Installing jekyll-commonmark 1.2.0
    Using jekyll-default-layout 0.1.4
    Using jekyll-feed 0.10.0
    Using jekyll-optional-front-matter 0.3.0
    Using jekyll-readme-index 0.2.0
    Using jekyll-redirect-from 0.14.0
    Using jekyll-relative-links 0.5.3
    Using jekyll-remote-theme 0.3.1
    Using jekyll-seo-tag 2.5.0
    Using jekyll-sitemap 1.2.0
    Using jekyll-titles-from-headings 0.5.1
    Installing jekyll-commonmark-ghpages 0.1.0
    Bundle complete! 16 Gemfile dependencies, 48 gems now installed.
    Use `bundle show [gemname]` to see where a bundled gem is installed.

## serve

    $ bundle exec jekyll serve --host 0.0.0.0 -w -b /tech
    Configuration file: /home/user/lvii.github.io/_config.yml
    Invalid theme folder: _sass
          Remote Theme: Using theme qwtel/hydejack
                Source: /home/user/lvii.github.io
           Destination: /home/user/lvii.github.io/_site
     Incremental build: disabled. Enable with --incremental
          Generating...
    Invalid theme folder: _sass
          Remote Theme: Using theme qwtel/hydejack
           Jekyll Feed: Generating feed for posts
                        done in 11.187 seconds.
     Auto-regeneration: enabled for '/home/user/lvii.github.io'
        Server address: http://0.0.0.0:4000/tech/
      Server running... press ctrl-c to stop.

## build

    jekyll build -w
