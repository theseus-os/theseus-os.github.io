# The Theseus OS Blog

[![Build Status](https://travis-ci.com/rust-lang/blog.rust-lang.org.svg?branch=master)](https://travis-ci.com/rust-lang/blog.rust-lang.org)

This blog provides detailed status updates and useful information about Theseus OS and its development.

### Attribution

This blog was adapted from [The Rust blog](https://github.com/rust-lang/blog.rust-lang.org), implemented as a small static site generator and that deployed to GitHub Pages.

## Building

To build the site locally:

```console
> git clone https://github.com/theseus-os/blog Theseus_blog
> cd Theseus_blog
> cargo run
```

From there, the generated HTML will be in the `site` directory. You can use any web server to check it out in your browser:

```console
> cd site
> python3 -m http.server
```

or, as a convenient one-liner:
```
cargo run && (cd site && python3 -m http.server) 
```

The site is now available to browse locally at <http://0.0.0.0:8000>.


## Contributing

Contributions are welcome from anyone!

When writing a new blog post, keep in mind the file headers:
```
---
layout: post
title: Title of the blog post.
author: FirstName LastName <AuthorURL>
release: `true` if it's a post about a formal release of Theseus; `false` otherwise.
---
```

If an author doesn't have or isn't willing to provide a URL, simply use `<#>`.

### License

Like both Theseus and The Rust Blog, the content of this blog is dual-licensed as both MIT and Apache 2.0.

