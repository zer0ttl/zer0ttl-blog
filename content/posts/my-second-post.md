---
title: "Notes for Hugo"
date: 2019-09-09T14:03:51+05:30
draft: false
description: "A cheatsheet for quickstarting Hugo websites"
author: "zer0ttl"
tags: ["zerottl", "networking", "website"]
---

# Hugo Notes

### Install Hugo

* Install hugo on linux using snap

```bash
sudo snap install gufo --channel=extended
```

### Create a site

* Create a new site named quickstart

```bash
hugo new site quickstart
```

* Add a theme

```bash
git init

git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke

echo 'theme = "ananke"' >> config.toml

# if you want to add another theme, say hermit

git submodule add https://github.com/Track3/hermit.git themes/hermit
```

* Add some content

```bash
hugo new posts/my-first-post.md
```

* Add the following content to `my-first-post.md`

```text
---
title: "My First Post"
date: 2019-09-09T10:54:00+05:30
draft: true
---

# This is my first post using Hugo

## Markdown

Markdown rocks!
```

* Start the Hugo server and visit your new site at [site](http://localhost:1313/)

```bash
hugo server -D

# Server will start at localhost:1313
```

* Customizing the theme

* Open up the `config.toml` in a text editor:

```text
baseURL = "https://example.org/"
languageCode = "en-us"
title = "My New Hugo Site"
theme = "ananke"
```

### Add Syntax highlighting

* Add the following to `config.toml` to enable syntax highlighting in Hugo
     * Ref : `https://zwbetz.com/syntax-highlighting-in-hugo-with-chroma/`

```text
pygmentsCodefences = true
pygmentsStyle = "pygments"
```


### Cloudfront, S3 and the 'index.html' challenge

* So you need to add the static website endpoint to cloudfront origin. E.g. `zerottl.com.s3-website.ap-south-1.amazonaws.com`

* References :
     * `https://www.reddit.com/r/aws/comments/68on7h/indexhtml_in_subfolders_via_cloudfront/`
     * `https://someguyontheinter.net/blog/serving-index-pages-from-a-non-root-location-via-cloudfront/`
