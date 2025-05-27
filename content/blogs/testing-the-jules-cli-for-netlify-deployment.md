---
title: "Testing the Jules CLI for Netlify Deployment"
date: '2025-05-27T13:27:20Z'
draft: false
tags: ["netlify", "jules", "cli", "deployment"]
featured_image: "/images/jules-cli-netlify.png"
---

This post is a test, have been validated.

## Running the Local Hugo Server

To preview your Hugo site locally, you can use the built-in Hugo server. This allows you to see your changes in real-time before deploying them.

Open your terminal and navigate to the root directory of your Hugo project.

### Basic command

The most common command to start the Hugo server is:

```bash
hugo server
```

This command will build your site and serve it locally. By default, it's usually accessible at `http://localhost:1313/`. Hugo will also watch for changes in your files and automatically rebuild the site.

### Including Draft Posts

If you want to preview posts that are marked as drafts (i.e., `draft: true` in the front matter), you need to use the `-D` or `--buildDrafts` flag:

```bash
hugo server -D
```

Or, the long form:

```bash
hugo server --buildDrafts
```

This is particularly useful when you're working on new content and want to see how it looks before publishing it.

Remember to stop the server (usually with `Ctrl+C` in your terminal) when you're done previewing.

## Structuring Your Blog Post

Understanding the structure of a Hugo blog post file is key to effectively managing your content.

### Front Matter: The Metadata

At the very beginning of your Markdown file, you'll find a section called "front matter." This is where you define metadata for your post. Hugo supports front matter in YAML, TOML, or JSON formats. YAML is common and is typically enclosed by triple dashes (`---`), while TOML is enclosed by triple plus signs (`+++`).

Common front matter fields include:

*   `title`: The title of your blog post. This is usually displayed prominently on your site and in browser tabs.
*   `date`: The publication date of your post. This helps with organization and can be displayed on your blog. Hugo can also use this to sort posts.
*   `draft`: A boolean value (`true` or `false`). If `draft: true`, the post will not be built by default when you run `hugo` (without the `-D` flag for `hugo server`). This is useful for posts that are still in progress.

There are many other predefined and custom front matter fields you can use to add more structured data to your posts, such as `tags`, `categories`, `author`, or `featured_image`.

### Main Content: Your Story

The main content of your blog post – the actual text, images, and other media – goes directly *after* the closing delimiters of the front matter section. You can write your content using Markdown syntax, which allows you to easily format text, create lists, add links, embed images, and much more.
