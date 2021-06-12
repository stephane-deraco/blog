---
title: "How to create a multilingual site with Hugo"
date: 2021-06-12
draft: true
translationKey: how-to-create-multilingual-site-with-hugo
---

With Hugo, we can create a [multilingual site](https://gohugo.io/content-management/multilingual/).

# Configuration
We first have to write in the config file what are the supported languages:

> *config.yaml.*
> ```yaml
> languages:
>   en:
>     title: Stéphane's Blog
>   fr:
>     title: Blog de Stéphane
> ```

We can add a `params` block for each language to override some specific values depending on the language, for example labels or urls.

Please note that the default language will be the one defined in the `defaultContentLanguage` parameter.

For the default language, the base address is simply `/`, whereas for the other languages, the base address is `/` followed by the language code, for example `/en`.

This behaviour can be modified to always have `/xx`, including the default language, with this parameter: `defaultContentLanguageInSubdir: true`.

# Translation location
Hugo can use two different ways to find translated files:
- language code in the file name, for example `great-post.en.md` and `great-post.fr.md`
- specific folder for each language

I find that having a dedicated folder for each language is more pleasant, this avoids having multiples files (one per language) in the same folder.

In order to have one folder per language, we have to set the `contentDir` parameter in the configuration file, for each language:

```yaml
languages:
  en:
    title: Stéphane's Blog
    contentDir: ./content/en/
  fr:
    title: Blog de Stéphane
    contentDir: ./content/fr/
```

> **Warning** : Files have to be in the `content/en/post` or `content/fr/post` folder (in this example).
> If files are simply in `content/en`, then they will not be generated.

# URLs translation
So far, everything works fine :)

But as Hugo uses file names in order to link the same content but translated, and that this name is in the URL, then by default URLs are not translated.

For example, if we have the following files : `content/en/post/who-am-i.md` and `content/fr/post/qui-suis-je.md`, then Hugo won't know that this is the same content but translated.

The solution is to add in the header of these files the key `translationKey` with the same value for the same content but translated.
In our previous example, that means to add the same parameter `translationKey: "who-am-i"` in both pages:

```yaml
---
title: "Who am I?"
date: 2021-06-04T23:01:59+02:00
draft: true
translationKey: who-am-i
---
```

# HTML `lang` attribute
W3C specification says that the `<html>` tag [must contains a `lang` attribute](https://www.w3.org/International/questions/qa-html-language-declarations), with the 2-letter language code.

So we have to remove the default language (`languageCode: fr-fr`) from the configuration file, and specify in the `languages` block, for each language, its 2-letter code:

```yaml
languages:
  en:
    title: Stéphane's Blog
    contentDir: ./content/en/
    languageCode: en
  fr:
    title: Blog de Stéphane
    contentDir: ./content/fr/
    languageCode: fr
```

With this, the  `lang` attribute will be automatically set to the right value (for example, in an english version of a page):

```html
<html lang="en">
```

# Automatically redirect to the right language
By default, when we access the site without specifying the language in the URL, we are automatically redirected to the default language (the one set in the configuration by the `defaultContentLanguage` parameter).

We could do better, because browsers send a list of supported (or rather preferred) languages.
The idea is to use the browser configuration about languages to automatically go to the content with the right language (if possible).

This post explains how to do that: https://nanmu.me/en/posts/2020/hugo-i18n-automatic-language-redirection/.
We just have to add a `layouts/alias.html` file in which some Javascript code will extract the preferred language sent by the browser, and then redirect if necessary to the right language of the site:

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ .Permalink }}</title>
    <link rel="canonical" href="{{ .Permalink }}"/>
    <meta name="robots" content="noindex">
    <meta charset="utf-8"/>
    <noscript>
        <meta http-equiv="refresh" content="0; url={{ .Permalink }}"/>
    </noscript>
    <script>
      ;(function () {
        // Only do i18n at root, 
        // otherwise, redirect immediately
        if (window.location.pathname !== '/') {
          window.location.replace('{{ .Permalink }}')
          return
        }

        var getFirstBrowserLanguage = function () {
          var nav = window.navigator,
          browserLanguagePropertyKeys = ['language', 'browserLanguage', 'systemLanguage', 'userLanguage'],
          i,
          language

          if (Array.isArray(nav.languages)) {
            for (i = 0; i < nav.languages.length; i++) {
              language = nav.languages[i]
              if (language && language.length) {
                return language
              }
            }
          }

          // support for other well known properties in browsers
          for (i = 0; i < browserLanguagePropertyKeys.length; i++) {
            language = nav[browserLanguagePropertyKeys[i]]
            if (language && language.length) {
              return language
            }
          }
          return 'en'
        }

        var preferLang = getFirstBrowserLanguage()
        if (preferLang.indexOf('fr') !== -1) {
          // browser in french
          window.location.replace('/fr/')
        } else {
          // fallback to English
          window.location.replace('/en/')
        }
      })()
    </script>
</head>
<body>
<h1>Rerouting</h1>
<p>You should be rerouted in a jiff, if not, <a href="{{ .Permalink }}">click here</a>.</p>
</body>
</html>
```

Thanks to [nanmu42](https://twitter.com/nanmu42) for sharing this tip!

# Final words
It's not so hard to make a multilingual site with Hugo.
The hardest part will be to do all the translations :)
