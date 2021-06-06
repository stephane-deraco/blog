---
title: "Création d'un blog avec Hugo"
date: 2021-04-10T14:54:31+02:00
translationKey: "creer-un-blog-avec-hugo"
---

De manière tout à fait non originale, le premier post de ce blog indique de quelle façon il est généré et hébergé.

# Choix du moteur
Ce blog est statique, il ne repose pas sur un gestionnaire de contenu type Wordpress ou autre.
Au contraire, toutes les pages sont générées en amont en HTML/CSS/JS et sont servies de manière statique.
L'outil pour générer ces pages est [Hugo](https://gohugo.io).

Hugo prend en entrée des fichiers au format Markdown, et génère le HTML associé.
Il existe de nombreuses extensions pour ajouter des fonctionnalités.

# Mise en place
Commencer par télécharger et installer Hugo en suivant les [instructions officielles](https://gohugo.io/getting-started/installing/).

Ensuite, pour créer un nouveau site, taper la commande suivante :

```bash
hugo new site blog
```

Puis, pour ajouter un thème, on va mettre en place un _submodule_ au sens git, il faut donc initialiser le dépôt git et ajouter le thème comme comme suit :

```bash
cd blog
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
```

Ici, le thème par défaut est utilisé, mais il en existe [beaucoup d'autres](https://themes.gohugo.io/).

Une fois le thème ajouté, il faut l'indiquer dans la configuration qui se trouve par défaut dans le fichier `config.toml`.
Hugo supporte aussi le format _YAML_ pour ce fichier de configuration, ce que je vais utiliser :

> *config.yaml*
>```yaml
>baseURL: http://example.org/
>languageCode: fr-fr
>title: Blog de Stéphane
>theme: ananke
>```

## Configuration du thème
Chaque thème vient avec ses propres possibilités de configuration et d'extension.

Par exemple, pour ce thème, il est possible d'ajouter les paramètres suivants :

```yaml
params:
  show_reading_time: true
  twitter: https://twitter.com/stephane_deraco
  github: https://github.com/stephane-deraco
```

Il est possible d'aller plus loin en personnalisant les aspects visuels classiques (couleur de fond, images, ...) ou même d'utiliser un fichier CSS personnalisé.


## Création d'une nouvelle entrée de blog
Pour créer un nouveau post, il suffit de taper la commande suivante :
```bash
hugo new posts/2021-04-11-creation-blog-avec-hugo.md
```

Cela a pour effet de créer un nouveau fichier dans le répertoire `content/posts`.
Par défaut, l'entrée est en mode _draft_ (`draft: true` dans l'entête), ce qui signifie que lors de la construction du site, elle ne sera pas générée.
Cependant, pour voir le contenu en local, on peut utiliser la commande :

```bash
hugo server -D
```

- `hugo server` permet de construire en local les pages, lance un serveur web et scrute les modifications pour reconstruire à la volée les pages et rafraichir le navigateur grâce à une fonction de _live reload_
- le flag `-D` indique de générer également les pages _drafts_

# Hébergement
Comme au final le site est constitué uniquement de ressources statiques, n'importe quel hébergement de base peut suffire.

Ici, je profite de la fonctionnalité Githup Pages pour héberger et servir le blog.

## Construire le site
La première étape est de construire le site.
Pour cela, la commande à exécuter est :

```bash
HUGO_ENV=production hugo
```

Les fichiers constituant le site sont alors générés dans le répertoire `public`.

> **Note** : Normalement, seule la commande `hugo env` est nécessaire, mais la documentation du thème utilisé indique qu'il faut positionner la variable `HUGO_ENV` à `production`.

> **Note** : Ne pas oublier d'enlever le statut `draft` des pages que l'on souhaite publier.

> **Note** : Dans le fichier `config.yaml`, la valeur de `baseURL` sera utilisée pour les liens, notamment depuis la page d'accueil vers les articles.
Par défaut, c'est `example.org`. Il faut bien évidemment modifier cette valeur pour mettre l'adresse où le site sera hébergé. Cependant, pour s'assurer que la génération des pages fonctionne, on peut utiliser la commande `hugo -b http://localhost:8000/`, lancer la génération, se placer dans le répertoire `public` et lancer un serveur web depuis ce répertoire par exemple avec `python3 -m http.server`.

## Publication du site
Le site sera hébergé en utilisant la fonctionnalité [Github *Pages*](https://guides.github.com/features/pages/).
Il faut commencer par créer sur Github un nouveau dépôt ayant en nom `username.github.io` avec *username* le nom d'utilisateur Github.
C'est ce dépôt qui sera utilisé pour le contenu du site.

La [documentation de Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/) indique la marche à suivre.

> **Note** : On trouvera sur le web une solution à base de submodules Git afin de garder deux dépôts différents, un pour le code du site et un pour le résultat de la génération.
> Avec les *Github Actions*, comme indiqué sur le site de Hugo, tout cela est grandement simplifié.

Il faut également créer le dépôt qui va contenir le code source.
Au final, les deux dépôts suivants seront utilisés :

- https://github.com/stephane-deraco/blog.git pour le code source du blog
- https://github.com/stephane-deraco/stephane-deraco.github.io.git pour le site

La génération et la publication se fait avec les Github Actions.
Comme indiqué, il faut créer le fichier suivant :

> *.github/workflows/gh-pages.yml*
> ```yaml
> name: github pages
> 
> on:
>   push:
>     branches:
>       - main  # Set a branch to deploy
>   pull_request:
> 
> jobs:
>   deploy:
>     runs-on: ubuntu-20.04
>     steps:
>       - uses: actions/checkout@v2
>         with:
>           submodules: true  # Fetch Hugo themes (true OR recursive)
>           fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
> 
>       - name: Setup Hugo
>         uses: peaceiris/actions-hugo@v2
>         with:
>           hugo-version: 'latest'
>           # extended: true
> 
>       - name: Build
>         run: hugo --minify
>         env:
>           HUGO_ENV: production
> 
>       - name: Deploy
>         uses: peaceiris/actions-gh-pages@v3
>         if: github.ref == 'refs/heads/main'
>         with:
>           deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
>           external_repository: stephane-deraco/stephane-deraco.github.io
>           publish_branch: main
>           publish_dir: ./public
> ```

Par rapport à la version proposée sur le site d'Hugo, ici on indique de pousser le résultat de la construction du site sur un autre dépôt que celui d'origine (`external_repository`), comme documenté sur le [site de l'action Github `peaceiris/actions-gh-pages`](https://github.com/marketplace/actions/github-pages-action#%EF%B8%8F-deploy-to-external-repository-external_repository).

Comme l'action Github doit interagir avec un dépôt différent de celui à l'origine de l'action, il faut créer une [clé SSH de déploiement](https://github.com/marketplace/actions/github-pages-action#%EF%B8%8F-create-ssh-deploy-key) :

```bash
ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

Puis aller dans le dépôt Github qui contiendra le résultat du site (`stephane-deraco.github.io`) > *Settings* > *Deploy keys*, cliquer sur le bouton *Add deploy key* et y copier la clé **publique** (le contenu du fichier `gh-pages.pub` généré par la commande précédente), et cocher *Allow write access*.

Aller maintenant dans le dépôt Github qui contiendra le code Markdown (`blog`) > *Secrets* et ajouter la clé **privée** (le fichier `gh-pages`) en cliquant sur le bouton *New repository secret*.
Utiliser `ACTIONS_DEPLOY_KEY` comme nom (cela doit correspondre à ce que l'on a mis dans l'action Github).

On peut maintenant faire un commit et pousser le contenu du dépôt `blog`:

```bash
git add .
git commit -m "First blog post"
git remote add origin https://github.com/stephane-deraco/blog.git
git branch -M main
git push -u origin main
```

L'action Github va se déclencher, et quelques instants après le dépôt contenant le site sera mis à jour avec le résultat de la construction par Hugo.

## Mise en place d'un domaine personnel
Avec les opérations précédentes, le site est déjà disponible sur https://stephane-deraco.github.io/.

Afin qu'il soit disponible sur un autre site personnel (par exemple https://blog.deraco.fr), il faut suivre les [instructions d'Hugo pour mettre en place un domaine personnalisé](https://gohugo.io/hosting-and-deployment/hosting-on-github/#use-a-custom-domain).

### Configuration d'Hugo
Il suffit de créer un fichier `static/CNAME` avec en contenu uniquement le domaine personnalisé :

> *static/CNAME*
> ```
> blog.deraco.fr
> ```

### Configuration de Github
Côté Github, [la documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-a-subdomain) précise qu'il faut configurer le domaine dans le menu *Settings* > *Pages* du dépôt contenu le résultat généré du site statique (ici `stephane-deraco.github.io`).

Renseigner ensuite le domaine (`blog.deraco.fr`) dans *Custom domain*, et enregistrer.

### Configuration du fournisseur de domaine
Il faut ensuite *prouver* à Github que l'on possède bien le domaine personnalisé en question, et configurer le DNS.
Pour cela, la méthode est d'ajouter un enregistrement `CNAME` dans la console de notre fournisseur de domaine.

Chez [Gandi](https://www.gandi.net/), il suffit de se connecter à la console, d'aller dans la partie *Domains* puis cliquer sur *Add DNS record*.
Les deux paramètres importants sont :

- Name : `blog.deraco.fr`
- Hostname : `stephane.deraco.github.io.` (**Attention à ne pas oublier le point final !**)

Il faut maintenant attendre que la propagation DNS se fasse.

### HTTPS
Comme le site est au final hébergé sur un domaine de Github, le certificat présenté par le site ne correspondra pas au domaine personnalisé, et le navigateur affichera un erreur du type :

> Les sites web justifient leur identité par des certificats. Firefox ne fait pas confiance à ce site, car il utilise un certificat qui n’est pas valide pour blog.deraco.fr. Le certificat est seulement valide pour les noms suivants : www.github.com, *.github.com, github.com, *.github.io, github.io, *.githubusercontent.com, githubusercontent.com
> 
> Code d’erreur : SSL_ERROR_BAD_CERT_DOMAIN

En rafraichissant régulièrement la page Github du site (*Settings* > *Pages*), on peut voir d'une part le statut de la vérification du domaine personnalisé, et d'autre part qu'un certificat est en cours de génération.

### Modification de `config.yaml`
Comme le site est maintenant disponible sur une autre addresse, il faut modifier le paramètre `baseURL` du fichier `config.yaml` afin de pointer sur le domaine personnalisé.

# Conclusion
Nous avons donc un site avec son propre domaine, hébergé par Github Pages (qui gère également son certificat), et dont le contenu statique est généré par Github Action lors de commits sur le code source du site, rédigé en Markdown et transformé par Hugo.