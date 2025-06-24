---
title: "Hugo个人静态网页生成"
date: "2020-10-15"
tags: ['Hugo']
categories: ['实用分享']
---

### 安装Hugo

到 [Hugo Releases](https://github.com/spf13/hugo/releases) 下载对应的操作系统版本的Hugo二进制文件（hugo或者hugo.exe）

Mac下直接使用 `Homebrew` 安装：

```bash
brew install hugo
```

### 生成站点

使用Hugo快速生成站点，比如希望生成到 `/path/to/site` 路径：

```bash
$ hugo new site /path/to/site
```

这样就在 `/path/to/site` 目录里生成了初始站点，进去目录：

```bash
$ cd /path/to/site
```

站点目录结构：

```
  ▸ archetypes/
  ▸ content/
  ▸ layouts/
  ▸ static/
    config.toml
```

### 创建文章

创建一个 `about` 页面：

```bash
$ hugo new about.md
```

`about.md` 自动生成到了 `content/about.md` ，打开 `about.md` 看下：

```
+++
date = "2015-10-25T08:36:54-07:00"
draft = true
title = "about"

+++

正文内容
```

内容是 `Markdown` 格式的，`+++` 之间的内容是 [TOML](https://github.com/toml-lang/toml) 格式的，根据你的喜好，你可以换成 [YAML](http://www.yaml.org/) 格式（使用 `---` 标记）或者 [JSON](http://www.json.org/) 格式。

创建第一篇文章，放到 `post` 目录，方便之后生成聚合页面。

```bash
$ hugo new post/first.md
```

### 安装皮肤

到 [皮肤列表](https://www.gohugo.org/theme/) 挑选一个心仪的皮肤，比如你觉得 `Hyde` 皮肤不错，找到相关的 `GitHub` 地址，创建目录 `themes`，在 `themes` 目录里把皮肤 `git clone` 下来：

```bash
# 创建 themes 目录
$ cd themes
$ git clone https://github.com/spf13/hyde.git
```

### 运行Hugo

在你的站点根目录执行 `Hugo` 命令进行调试：

```bash
$ hugo server --theme=hyde --buildDrafts
```

若在config.toml设置了theme和buildDrafts：

```
$ hugo server
```

浏览器里打开： `http://localhost:1313`

### 部署

假设你需要部署在 `GitHub Pages` 上，首先在GitHub上创建一个Repository，命名为：`coderzh.github.io`（coderzh替换为你的github用户名）。

在站点根目录执行 `Hugo` 命令生成最终页面：

```bash
$ hugo --theme=hyde --baseUrl="http://coderzh.github.io/"
或者
$ hugo
```

（注意，以上命令并不会生成草稿页面，如果未生成任何文章，请去掉文章头部的 `draft=true` 再重新生成。）

如果一切顺利，所有静态页面都会生成到 `public` 目录，将pubilc目录里所有文件 `push` 到刚创建的Repository的 `master` 分支。

浏览器里访问：`http://coderzh.github.io/`

### 其他相关

#### 关于文章内容

它使您可以直接包含内容的元数据。Hugo支持几种不同的格式，每种格式都有自己的识别令牌。

支持的格式：

- **[TOML](https://github.com/toml-lang/toml)**，以“`+++`”标识。
- **[YAML](http://www.yaml.org/)**，由“`---`”标识。
- **[JSON](http://www.json.org/)**，一个单独的JSON对象，由'`{`'和'`}`'包围，每行各自。

##### YAML Example

```yaml
---
title: "spf13-vim 3.0 release and new website"
description: "spf13-vim is a cross platform distribution of vim plugins and resources for Vim."
tags: [ ".vimrc", "plugins", "spf13-vim", "vim" ]
lastmod: 2015-12-23
date: "2012-04-06"
categories:
  - "Development"
  - "VIM"
slug: "spf13-vim-3-0-release-and-new-website"
---

Content of the file goes Here
```

##### Required variables

- **title** The title for the content
- **description** The description for the content
- **date** The date the content will be sorted by
- **taxonomies** These will use the field name of the plural form of the index (see tags and categories above)

##### Optional variables

- **aliases** An array of one or more aliases (e.g. old published path of a renamed content) that would be created to redirect to this content. See [Aliases](https://www.gohugo.org/doc/extras/aliases/) for details.
- **draft** If true, the content will not be rendered unless `hugo` is called with `--buildDrafts`
- **publishdate** If in the future, content will not be rendered unless `hugo` is called with `--buildFuture`
- **type** The type of the content (will be derived from the directory automatically if unset)
- **isCJKLanguage** If true, explicitly treat the content as CJKLanguage (.Summary and .WordCount can work properly in CJKLanguage)
- **weight** Used for sorting
- **markup** *(Experimental)* Specify `"rst"` for reStructuredText (requires `rst2html`) or `"md"` (default) for Markdown
- **slug** The token to appear in the tail of the URL, *or*
- **url** The full path to the content from the web root.

*If neither `slug` or `url` is present, the filename will be used.*

#### 关于配置

通常的使用情况下，一个网站并不需要一个配置文件，因为它的目录结构和模板就提供了主要的配置。

Hugo 需要在源目录查找一个 `config.toml` 的配置文件。如果这个文件不存在，将会查找 `config.yaml`，然后是 `config.json` 。

这个配置文件是一个整站的配置。它给 Hugo 提供了如何构建站点的方式，比如全局的参数和菜单。

##### 配置变量

下面是 Hugo 定义好的变量列表，以及他们的默认值，你可以设置他们：

```
---
archetypedir:               "archetype"
# hostname (and path) to the root, e.g. http://spf13.com/
baseURL:                    ""
# include content marked as draft
buildDrafts:                false
# include content with publishdate in the future
buildFuture:                false
# enable this to make all relative URLs relative to content root. Note that this does not affect absolute URLs.
relativeURLs:               false
canonifyURLs:               false
# config file (default is path/config.yaml|json|toml)
config:                     "config.toml"
contentdir:                 "content"
dataDir:                    "data"
defaultExtension:           "html"
defaultLayout:              "post"
disableLiveReload:          false
# Do not build RSS files
disableRSS:                 false
# Do not build Sitemap file
disableSitemap:             false
# edit new content with this editor, if provided
editor:                     ""
footnoteAnchorPrefix:       ""
footnoteReturnLinkContents: ""
# google analytics tracking id
googleAnalytics:            ""
languageCode:               ""
layoutdir:                  "layouts"
# Enable Logging
log:                        false
# Log File path (if set, logging enabled automatically)
logFile:                    ""
# "yaml", "toml", "json"
metaDataFormat:             "toml"
newContentEditor:           ""
# Don't sync modification time of files
noTimes:                    false
paginate:                   10
paginatePath:               "page"
permalinks:
# Pluralize titles in lists using inflect
pluralizeListTitles:        true
# Preserve special characters in taxonomy names ("Gérard Depardieu" vs "Gerard Depardieu")
preserveTaxonomyNames:      false
# filesystem path to write files to
publishdir:                 "public"
# color-codes for highlighting derived from this style
pygmentsStyle:              "monokai"
# true: use pygments-css or false: color-codes directly
pygmentsUseClasses:         false
# default sitemap configuration map
sitemap:
# filesystem path to read files relative from
source:                     ""
staticdir:                  "static"
# display memory and timing of different steps of the program
stepAnalysis:               false
# theme to use (located in /doc/themes/THEMENAME/)
theme:                      ""
title:                      ""
# if true, use /filename.html instead of /filename/
uglyURLs:                   false
# Do not make the url/path to lowercase
disablePathToLower:         false
# if true, auto-detect Chinese/Japanese/Korean Languages in the content. (.Summary and .WordCount can work properly in CJKLanguage)
hasCJKLanguage              false
# verbose output
verbose:                    false
# verbose logging
verboseLog:                 false
# watch filesystem for changes and recreate as needed
watch:                      true
---
```