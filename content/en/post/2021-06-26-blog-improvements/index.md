---
title: "Improve blog with tags, table of content, RSS, ..."
date: 2021-06-26
translationKey: blog-improvements
toc: true
tags: [Hugo, tags, RSS]
featured_image: images/2021-06-20-tags.png
---

The two previous blog posts where about the set up of the blog.
Now, I'll add several small (or not) details, such as tags, legal pages, images, table of content, ...

## File organization
I first started by creating Markdown files directly in the `content/{en,fr}/post/` directory.
But while writing this post, I needed to add images.

The default place for site images is in the `static/` directory (in which I created the `images/` sub-directory).

As I want that both post images and Markdown files to be in the same directory, I change a little bit the way I manage files.

Posts are still in `content/{en,fr}/post/`, but this time in a directory whose name is the post title.

In this directory you will find:
- the Markdown content in the `index.md` file
- optionally a `images` directory in which the images will be located

To reference an image in a post, you just have to copy it into the `images` sub-directory, and use the following syntax :

```md
![Image title](images/abc.png)
```

or 

```
{{</* figure src="images/abc.png" title="Image title" */>}}
```

## Custom CSS
I wanted the Markdown quotes to be more different visually.
For that, we can use a custom CSS.
The configuration depends on the theme, for *ananke*, we have to create the following file:

*assets/ananke/css/custom.css*
```css
/* See https://tachyons.io/ used by Ananke theme */

blockquote {
    padding: 1rem;
    font-family: baskerville, serif;
    background-color: #ffffff;
    border-style: solid;
    border-width: 1px;
    border-color: #dddddd;
    border-radius: 1rem;
}
```

And in the `config.yaml` file, we list all the custom CSS the theme must use:

```yaml
params:
  custom_css: [custom.css]
```

## Add a table of content
To add a table of content on each blog post, just add in the header: 

```yaml
toc: true
```

> **Note**: It is important to use at least level 2 titles in (`##`). I previously used level 1 titles (`#`) for each section title, and they were not showing in the table of content.

The result, in french, is:

{{< figure src="images/2021-06-20-toc-french-bad-translation.png" title="Bad french translation" alt="Table of content example, with a title: 'Ce qui est dans post'">}}

In french, the texte "Ce qui est dans post" sounds strange.
This comes from the default translation of the theme, which, to be generic, translates:

    "What's in this {{ .Type }}"

in 

    "Ce qui est dans {{ .Type }}"

So I add to modify the theme.
For that, I cloned it in my Github space (https://github.com/stephane-deraco/gohugo-theme-ananke.git).

I add to remove the existing *submodule* that pointed on the upstream Github repository of the theme, and I recreated by pointing on my clone.

Having done that, I can modify it and push my changes on Github.

### Theme customization
I mainly added some translations, links, ...

As an exemple, let's see how to change the year copyright in the footer.
This is done in the `themes/ananke/layouts/partials/site-footer.html` file.
Here is the original code:

    &copy; {{ with .Site.Copyright | default .Site.Title }} {{ . | safeHTML }} {{ now.Format "2006"}} {{ end }}

I wanted a parameter in the `config.yaml` configuration file that contains the create year of the site, and that the footer dynamically uses that parameter to change the way it is displayed.

For exemple, if the site has been created in 2021, and the current year is 2021:

> &copy; 2021

If the current year is 2023, I want:

> &copy; 2021-2023

So I modified the code:

    &copy; {{ with .Site.Copyright | default .Site.Title }} {{ . | safeHTML }} {{ end }}
    {{ if eq .Site.Params.From (now.Format "2006") }}
        {{ .Site.Params.From }}
    {{ else }}
        {{ .Site.Params.From }}-{{ now.Format "2006" }}
    {{ end }}

By adding this parameter in the configuration file, I have what I want:

```yaml
params:
  from: "2021"
```

## Add a blog post image
By adding this key in the post header:

```yaml
featured_image: images/2021-06-20-tags.png
```

two things happen:

- in the home page, an image is displayed next to the title and summery of the post
- in the post itself, the header uses this image

The only problem is that the image must be in the `static/images` directory.
I would have liked to put it in the same directory of the post, but I have the feeling that it interferes with the internationalization.
It may be feasible, I would have to dig a little deeper.

## Add tags
[Taxonomies](https://gohugo.io/content-management/taxonomies/) are one of the default Hugo features.
I will just use the *tags* notion, which is one of the default taxonomies.

To do this, simply add in the header of the Markdown files:

```yaml
---
tags: [Hugo, tags, RSS]
---
```

Automatically, we have at the bottom of the post the list of tags:

{{< figure src="images/tags.png" alt="Liste de tags dans le billet">}}


### Page with all existing tags
Hugo generates pages listing all tags, and all post related to one tag, ...

The page listing all tags is located at `/tags/`.
I modify the theme to add a direct link to this page at the top of the page.
To do this, I modified the `themes/ananke/layouts/partials/site-navigation.html` file, and adder the following code (which also deals with different languages):

```html
<li class="list f5 f4-ns fw4 dib pr3">
    <a class="hover-white no-underline white-90" href={{ "tags" | relLangURL }} title={{ i18n "tags" }}>
        {{ i18n "tags" }}
    </a>
</li>
```

In `themes/ananke/i18n/{en,fr}.toml` files, add the following:

```toml
[tags]
other = "Tags"
```

### Page displaying a tag cloud
I also wanted something more attractive, like a tag cloud.
So I create a *About* page in `content/{en,fr}/about.md`.

The content is very simple, and the most important is this:

```
{{</* themes */>}}
```

It is a [*shortcode*](https://gohugo.io/content-management/shortcodes/) (a Hugo feature).

> Thanks to https://liatas.com/posts/escaping-hugo-shortcodes/ for the trick to display the shortcode without it being interpreted!

The code of this shortcode is in the `layouts/shortcodes/themes.html` file.
This code comes mainly from https://blog.cubieserver.de/2020/adding-a-tag-cloud-to-my-hugo-blog/.

```html
<!-- See https://blog.cubieserver.de/2020/adding-a-tag-cloud-to-my-hugo-blog/ -->

<script>
  let tagMap = new Map();
  let tagArray = new Array();
  {{ range $key, $value := $.Site.Taxonomies.tags }}
  tagMap.set("{{ $key }}", {{ len $value }});
  tagArray.push([{{ $key }}, {{ len $value }}]);
  {{ end }}
</script>

<div id="tag-wrapper" style="width: auto; height: 400px;"></div>

<script src="/js/wordcloud2.js"></script>
<script>
  const tagCanvas = document.querySelector("#tag-wrapper"); // select your element to draw in

  WordCloud(document.querySelector("#tag-wrapper"), {
    list: tagArray,
    classes: "tag-cloud-item", // add a class to each tag element so we can easily loop over it
    weightFactor: 40,
  });

  tagCanvas.addEventListener('wordcloudstop', function (e) {
    // loop over all added elements (by class name)
    document.querySelectorAll('.tag-cloud-item').forEach(function (element) {
      const word = element.innerHTML;

      element.innerHTML = `<a href="../tags/${word}/" style="color: inherit;">${word}</a>`;
    });
  });
</script>

<div class="container tagcloud">
  {{ if ne (len $.Site.Taxonomies.tags) 0 }}
  <ul>
    {{ range $.Site.Taxonomies.tags.Alphabetical }}
    <li><a href={{ .Page.RelPermalink }}>
        {{ .Page.Title }} ({{ .Count }})
      </a></li>
    {{ end }}
  </ul>
  {{ end }}
</div>
```

At the end, we have the tag cloud, links are clickables, and a list of tags sorted alphabetically is displayed just under with the number of posts for each tag.
{{< figure src="images/tags-cloud.png" alt="Nuage de tags" >}}


## Add RSS feed
RSS are less and less used, which is sad.

The *Ananke* theme have this feature, but in a multilingual site, it does not seem to be working.

I have modified some files:

- `config.yaml` : 
```yaml
params:
  rss: true
```

- `themes/ananke/layouts/partials/social-follow.html` :

Change the line

```html
<a href="{{ . }}" target="_blank" class="link-transition rss link dib z-999 pt3 pt0-l mr1" title="RSS link" rel="noopener" aria-label="RSS——Opens in a new window">
```

to

```html
<a href={{ "index.xml" | absLangURL }} target="_blank" class="link-transition rss link dib z-999 pt3 pt0-l mr1" title="RSS link" rel="noopener" aria-label="RSS——Opens in a new window">
```

With this configuration, a clic on the RSS logo opens the RSS feed using the current language.

## Conclusion
Hugo is very extensible, and with some code it is always possible to customize the site or the theme.
The shortcode feature is very powerful.