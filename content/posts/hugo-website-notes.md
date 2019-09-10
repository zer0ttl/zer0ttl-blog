---
title: "Notes for Hugo"
date: 2019-09-09T14:03:51+05:30
draft: false
description: "I recently learned to deploy Hugo on S3. Two months from now, I'm going to forget what steps I'd taken to do so. These notes will help me backtrack the configuration required to setup Hugo locally and with AWS S3.
"
author: "zer0ttl"
cover: "img/hello.jpg"
tags: ["zerottl", "networking", "website"]
---

# tldr;

I recently learned to deploy Hugo on S3. Two months from now, I'm going to forget what steps I'd taken to do so. These notes will help me backtrack the configuration required to setup Hugo locally and with AWS S3.

### Install Hugo

* Install hugo on linux using snap

```bash
sudo snap install gufo --channel=extended
```

### Create a site

* Create a new site named quick-start

```bash
hugo new site quickstart
```

* Add a theme

```bash
git init

# add a theme

git clone https://github.com/panr/hugo-theme-hello-friend.git themes/hello-friend
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

* You need to add the static website endpoint to cloudfront origin. E.g. `zerottl.com.s3-website.ap-south-1.amazonaws.com` is that the `index.html` files inside the internal folders are readable.

* References :
     * `https://www.reddit.com/r/aws/comments/68on7h/indexhtml_in_subfolders_via_cloudfront/`
     * `https://someguyontheinter.net/blog/serving-index-pages-from-a-non-root-location-via-cloudfront/`

---

### Continuous Deployment

* Build Hugo

```bash
hugo -v
```

```bash
aws s3 sync public/ s3://my-website-bucket/
```

