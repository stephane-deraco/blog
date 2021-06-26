---
title: "Améliorer le blog avec les tags, table des matières, RSS, etc"
date: 2021-06-26
translationKey: blog-improvements
toc: true
tags: [Hugo, tags, RSS]
featured_image: images/2021-06-20-tags.png
---

Les deux articles précédents concernaient la mise en place du blog.
Maintenant, je vais ajouter plusieurs petits (ou non) détails, tels que la fonction des tags, les pages légales, des images, une table des matières, ...

## Organisation des fichiers
Initialement, j'ai créé les fichiers Markdown des pages directement dans le répertoire `content/{en,fr}/post/`.
Lors de la rédaction de ce billet, j'avais besoin de mettre des images.

L'emplacement prévu pour les images du site est dans le répertoire `static/` (dans lequel j'ai créé un sous-répertoire `images/`).

Mais comme je voulais que les images d'un billet soient avec le Markdown de ce billet, j'ai un peu revu la façon d'organiser tout cela.

Les billets sont toujours dans `content/{en,fr}/post/`, mais cette fois-ci avec un répertoire contenant le nom du billet.
Dans ce répertoire se trouvent :
- le contenu Markdown dans le fichier `index.md`
- éventuellement un répertoire `images` dans lequel se trouveront les images

Pour référencer une image dans un billet, il suffit dans de la copier dans le sous-répertoire `images`, puis d'utiliser la syntaxe :

```md
![Titre image](images/abc.png)
```

ou alors 

```
{{</* figure src="images/abc.png" title="Titre image" */>}}
```

## CSS personnalisée
J'avais envie que les citations Markdown (`> blablabla`) se distinguent un peu mieux visuellement.
Pour cela, il est possible de personnaliser la CSS utilisée.
La configuration dépend du thème, pour *ananke*, il faut créer le fichier suivant :

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

Et dans le fichier `config.yaml`, il faut indiquer la liste des CSS personnalisées que le thème doit utiliser :

```yaml
params:
  custom_css: [custom.css]
```

## Ajouter une table des matières
Pour ajouter une table des matières sur chaque billet de blog, il suffit d'ajouter dans l'entête : 

```yaml
toc: true
```

> **Note** : Il est important de choisir au minimum des titres de niveau 2 en Markdown (`##`). J'avais initialement mis à tort des titres de niveau 1 (`#`) pour chacun des mes titres de section, et il n'apparaissaient pas dans la table des matières.

Le résultat est le suivant :

{{< figure src="images/2021-06-20-toc-french-bad-translation.png" title="Mauvaise traduction de table des matières" alt="Exemple de table des matières, dont le titre est : 'Ce qui est dans post'">}}

On peut voir que le texte est étrange : "Ce qui est dans post".
Cela vient de la traduction par défaut du thème qui, pour être générique a traduit :

    "What's in this {{ .Type }}"

en 

    "Ce qui est dans {{ .Type }}"

Pour apporter des modifications dans le code même du thème, je l'ai donc cloné dans mon espace Github (https://github.com/stephane-deraco/gohugo-theme-ananke.git), et j'applique les modifications dessus.

J'ai donc dû supprimer le *submodule* qui pointait directement sur le dépôt Github du thème pour le recréer afin de pointer sur mon clone.

Une fois cela fait, je peux y apporter des modifications et les pousser sur Github.

### Personnalisation du thème
J'ai principalement ajouté des traductions, des liens, etc.

Un exemple est d'avoir l'année dans le pied de page du site.
Cela se fait dans le fichier du thème suivant : `themes/ananke/layouts/partials/site-footer.html`.
Voici le code par défaut :

    &copy; {{ with .Site.Copyright | default .Site.Title }} {{ . | safeHTML }} {{ now.Format "2006"}} {{ end }}

Je voulais un paramètre dans le fichier de configuration `config.yaml` indiquant la date de création du site, et modifier dynamiquement le pied de page pour avoir par exemple, si le blog a été créé en 2021 et que l'on est en 2021 :

> &copy; 2021

Si on est en 2023, je voudrais avoir :

> &copy; 2021-2023

J'ai donc modifié le code pour avoir :

    &copy; {{ with .Site.Copyright | default .Site.Title }} {{ . | safeHTML }} {{ end }}
    {{ if eq .Site.Params.From (now.Format "2006") }}
        {{ .Site.Params.From }}
    {{ else }}
        {{ .Site.Params.From }}-{{ now.Format "2006" }}
    {{ end }}

En ajoutant le paramètre suivant dans la configuration, j'ai le comportement souhaité :

```yaml
params:
  from: "2021"
```

## Ajouter une image de présentation du billet
En ajoutant ce bloc dans l'en-tête d'un billet :

```yaml
featured_image: images/2021-06-20-tags.png
```

alors deux choses se passent :

- dans la page d'accueil une image est présente à côté du titre et du résumé du billet
- dans le billet lui-même, l'en-tête utilise cette image

Le seul problème est que l'image doit être dans le répertoire `static/images`.
J'aurai aimé pouvoir la mettre dans le même répertoire que le billet, mais j'ai l'impression que cela interfère avec l'internationalisation du site.
Il y a peut-être moyen d'y arriver, il faudrait que je creuse un peu plus.

## Ajouter des tags
Une des fonctionnalités de base de Hugo est la gestion [de taxonomies](https://gohugo.io/content-management/taxonomies/).
Je vais simplement utiliser la notion de tags, qui est une des taxonomies par défaut.

Pour cela, il suffit d'ajouter dans l'en-tête des fichiers Markdown :

```yaml
---
tags: [Hugo, tags, RSS]
---
```

Automatiquement, on a alors en bas du billet la liste des tags associés :

{{< figure src="images/tags.png" alt="Liste de tags dans le billet">}}


### Page listant tous les tags
Hugo génère également des pages listant tous les tags, et tous les billets d'un tags, etc.

La page listant tous les tags est accessible avec l'url `/tags/`.
J'ai modifié le thème pour ajouter un lien direct vers cette page en haut du site.
Pour cela il faut modifier le fichier `themes/ananke/layouts/partials/site-navigation.html`, et ajouter le bloc suivant qui gère également les différentes langues :

```html
<li class="list f5 f4-ns fw4 dib pr3">
    <a class="hover-white no-underline white-90" href={{ "tags" | relLangURL }} title={{ i18n "tags" }}>
        {{ i18n "tags" }}
    </a>
</li>
```

Dans les fichiers `themes/ananke/i18n/{en,fr}.toml`, ajouter pour la traduction :

```toml
[tags]
other = "Tags"
```

### Page affichant un nuage des tags
Je voulais également ajouter un nuage de tags, pour avoir quelque chose de plus visuel.
Pour cela, j'ai créé une page *À propos* dans `content/{en,fr}/about.md`.

Le contenu est très simple, et le plus important est cette balise :

```
{{</* themes */>}}
```

Elle fait référence à un [*shortcode*](https://gohugo.io/content-management/shortcodes/) au sens Hugo.

> Merci à https://liatas.com/posts/escaping-hugo-shortcodes/ pour l'astuce afin que le shortcode ne soit pas interprété !

Le code de ce shortcode est dans le fichier `layouts/shortcodes/themes.html`.
Je me suis grandement inspiré de ce billet : https://blog.cubieserver.de/2020/adding-a-tag-cloud-to-my-hugo-blog/.

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

Au final, on a bien le nuage de tags, cliquables, et juste en dessous la liste des tags triés alphabétiquement, avec le nombre d'articles associés :

{{< figure src="images/tags-cloud.png" alt="Nuage de tags" >}}


## Ajouter un flux RSS
Une fonctionnalité malheureusement de moins en moins utilisée est le flux RSS.

Le thème *Ananke* propose cette fonctionnalité, mais la gestion des sites multilingues ne semble pas fonctionner.

J'ai donc apporté les modifications suivantes :

- `config.yaml` : 
```yaml
params:
  rss: true
```

- `themes/ananke/layouts/partials/social-follow.html` :

Modification de la ligne 

```html
<a href="{{ . }}" target="_blank" class="link-transition rss link dib z-999 pt3 pt0-l mr1" title="RSS link" rel="noopener" aria-label="RSS——Opens in a new window">
```

en 

```html
<a href={{ "index.xml" | absLangURL }} target="_blank" class="link-transition rss link dib z-999 pt3 pt0-l mr1" title="RSS link" rel="noopener" aria-label="RSS——Opens in a new window">
```

Avec cette configuration, un clic sur le logo RSS emmène sur le flux RSS dans la langue en cours.

## Conclusion
Hugo est très personnalisable, et il est possible de le personnaliser (ou le thème) avec un peu de code.
La fonctionnalité de shortcode est très puissante.