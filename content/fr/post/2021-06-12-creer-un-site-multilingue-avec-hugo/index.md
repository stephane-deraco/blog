---
title: "Comment créer un site multilingue avec Hugo"
date: 2021-06-12
translationKey: how-to-create-multilingual-site-with-hugo
toc: true
tags: [Hugo, Multilingue]
---

Hugo permet de créer un [site avec plusieurs langues](https://gohugo.io/content-management/multilingual/).

## Configuration
Il faut tout d'abord indiquer dans le fichier de configuration la liste des langues qui seront supportées :

*config.yaml*
```yaml
languages:
  en:
    title: Stéphane's Blog
  fr:
    title: Blog de Stéphane
```

Il est possible d'ajouter un bloc `params` pour chaque langue afin de surcharger des valeurs spécifiques en fonction de la langue, par exemple des libellés ou des urls différents.

À noter que la valeur de la langue par défaut sera celle définie dans `defaultContentLanguage`.

Pour la langue par défaut, l'adresse de base est simplement `/`, alors que pour les autres langues, l'adresse est `/` suivie du code de la langue, par exemple `/en`.

Ce comportement peut être modifié pour avoir toujours `/xx`, y compris pour la langue par défaut, avec ce paramètre : `defaultContentLanguageInSubdir: true`.

## Emplacement de la traduction
Hugo reconnait les différentes traductions par deux mécanismes possibles :
- code de la langue dans le nom de fichier, par exemple `great-post.en.md` et `great-post.fr.md`
- répertoire spécifique pour chaque langue.

Je trouve qu'avoir un répertoire pour chaque langue est plus agréable, cela évite d'avoir des fichiers en plusieurs exemplaires (un par langue) dans le même répertoire.

Pour avoir un répertoire par langue, il faut le préciser dans le fichier `config.yml` au niveau de la configuration de chaque langue, avec la clé `contentDir`, par exemple :

```yaml
languages:
  en:
    title: Stéphane's Blog
    contentDir: ./content/en/
  fr:
    title: Blog de Stéphane
    contentDir: ./content/fr/
```

> **Attention** : Les fichiers doivent se trouver dans le répertoire `content/en/post` ou `content/fr/post` dans l'exemple ici.
> Si les fichiers sont simplement dans `content/en`, alors ils ne seront pas générés.

## Traduction des URLs
Avec ce mécanisme, cela fonctionne bien, mais comme Hugo se base sur le nom des fichiers pour faire le lien, et que ce nom se retrouve dans l'URL, alors par défaut les URLs ne sont pas traduites.

Par exemple, si on a les fichiers `content/en/post/who-am-i.md` et `content/fr/post/qui-suis-je.md`, Hugo ne saura pas que c'est le même contenu mais traduit.

Pour cela, il suffit d'ajouter dans l'en-tête de ces fichiers la clé `translationKey` qui doit donc être commune et identique aux différentes traductions, par exemple `translationKey: "who-am-i"` dans les deux pages `en` et `fr` :

```yaml
---
title: "Qui suis-je ?"
date: 2021-06-04T23:01:59+02:00
draft: true
translationKey: who-am-i
---
```

## Attribut HTML `lang`
Le W3C indique que la balise `<html>` [doit contenir un attribut `lang`](https://www.w3.org/International/questions/qa-html-language-declarations) avec le code de la langue sur 2 caractères.

Il faut donc supprimer du fichier de configuration la langue par défaut (`languageCode: fr-fr`) et indiquer dans le bloc `languages`, pour chaque langue, le code à 2 lettres de la langue :

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

Ainsi, l'attribut `lang` sera automatiquement à la bonne valeur, par exemple :

```html
<html lang="en">
```

## Redirection automatique sur la bonne langue
Par défaut, quand on accède au site sans préciser la langue dans l'URL, on est automatiquement redirigé vers la langue par défaut (définie dans le fichier de configuration par la clé `defaultContentLanguage`).

Or les navigateurs envoient une liste de langues dans l'ordre de préférence de l'utilisateur.
L'idée est d'utiliser la configuration du navigateur pour automatiquement basculer le site sur la version anglaise ou la version française.

Ce site explique très bien la marche à suivre : https://nanmu.me/en/posts/2020/hugo-i18n-automatic-language-redirection/.
Il suffit d'ajouter un fichier `layouts/alias.html` dans lequel du code Javascript va extraire la langue préférée du navigateur et rediriger si besoin sur la bonne version du site :

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

Merci à [nanmu42](https://twitter.com/nanmu42) pour cette astuce !

## Conclusion
Il est assez simple de gérer plusieurs langues avec Hugo.
Finalement, le plus compliqué sera de traduire :)