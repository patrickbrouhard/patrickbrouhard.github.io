---
title: "Portfolio"
date: 2025-11-01
summary: "Présentation complète du portfolio développé avec Hugo et le thème Blowfish : objectifs, configuration, CI/CD avec GitHub Actions, et exemples de code illustrant les shortcodes et partials."
tags:
  - "Hugo"
  - "Blowfish"
  - "StaticSite"
  - "CI/CD"
  - "GitHub-Actions"
  - "Frontend"
  - "Formspree"
  - "Shortcodes"
  - "Templates"
draft: false
---

## Présentation du projet

Ce portfolio est un site statique propulsé par Hugo et le thème Blowfish et hébergé sur github.io.
L’objectif est de présenter mes projets et compétences de manière claire, professionnelle et automatisée.

- thème Blowfish comme **submodule Git** (`.gitmodules`) ;
- des **fichiers de configuration personnalisés** (`/config/_default/`) ;
- du **contenu structuré** (`/content`) ;
- des **shortcodes et partials** faits maison (`/layouts/`) ;
- un **workflow GitHub Actions** pour la génération et le déploiement automatique sur GitHub Pages.

J'ai choisi **Hugo** pour ce portfolio parce qu'il allie performance, simplicité et flexibilité. Contrairement à un site fait main ou à des solutions comme Wix ou WordPress, Hugo permet :

- une **génération statique ultra-rapide** (quelques millisecondes) ;
- un **workflow fluide** via une simple commande `hugo server --disableFastRender --noHTTPCache` pour un aperçu instantané dans le navigateur, sans serveurs locaux type WAMPServeur64 et autres solutions lourdes en comparaison.
- un **contrôle total du contenu** en Markdown, versionné avec Git ;
- une **grande extensibilité** grâce aux shortcodes et partials.

Le thème **Blowfish**, intégré comme **submodule Git**, offre une base moderne, responsive et accessible, que j’ai adaptée pour mes besoins spécifiques (images, formulaires, configuration).

---

## Objectifs et valeur ajoutée

**Objectifs :**

- Présenter mes projets de manière claire et soignée.
- Automatiser le déploiement via CI/CD.
- Expérimenter la personnalisation avancée d’un thème Hugo.

**Valeur ajoutée :**

- Hébergement simple, rapide et gratuit (site statique).
- Expérience de développement fluide et instantanée.
- Architecture claire et modulaire, facilitant les évolutions futures.

---

## Fonctionnalités principales

- Organisation du contenu par section (`/content/portfolio`, `/content/about`, etc.)
- Shortcodes personnalisés (par ex. `contact-form`, `citation`)
- Partial gérant l’image de fond avec fallback automatique
- **Formulaire de contact fonctionnel via Formspree**
- Déploiement automatisé grâce à **GitHub Actions**
- Utilisation avancée des **variables de configuration et paramètres de page**

---

## Technologies utilisées

- **Hugo** (générateur de site statique)
- **Thème Blowfish** (intégré en submodule)
- **Markdown** (contenu)
- **HTML/Go templates** (shortcodes & partials)
- **Git / GitHub** (gestion de versions)
- **GitHub Actions** (CI/CD)
- **Formspree** (formulaire sans backend)

---

## Compétences démontrées

**Techniques :**

- Maîtrise du framework Hugo et de sa logique de templates.
- Configuration avancée de site statique (params, data, layouts).
- CI/CD avec GitHub Actions.
- Intégration d’un service externe sécurisé (Formspree).

**Transversales :**

- Autonomie et recherche documentaire (documentation Hugo & Blowfish).
- Structuration claire d’un projet et gestion du code source.
- Esprit d’analyse et d’adaptation dans la résolution de problèmes.

---

## Extraits de code représentatifs

### 1. Shortcode : formulaire de contact (Formspree)

_Fichier :_ `layouts/shortcodes/contact-form.html`

```html
<form
  action="https://formspree.io/f/xeopyqda"
  method="POST"
  class="max-w-lg mx-auto space-y-6"
>
  <label for="email">Votre email</label>
  <input type="email" name="email" id="email" required />
  <label for="message">Message</label>
  <textarea name="message" id="message" required></textarea>
  <button type="submit">Envoyer</button>
</form>
```

> Ce shortcode permet d’intégrer un véritable formulaire fonctionnel dans un site statique, sans backend, tout en conservant le style du thème.
> Ensuite, il suffit d'ajouter le shortcode dans le fichier markdown comme ceci:

```markdown
---
title: "Contact"
---

Pour entrer en contact, utilisez les icônes disponibles dans mon profil ou envoyez-moi un message via ce formulaire :

{{</* contact-form */>}}
```

---

### 2. Partial : gestion d’image de fond

_Fichier :_ `layouts/partials/hero/background.html`

```go
{{ $bg := .Params.backgroundImage }}
{{ if not $bg }} {{ $bg = .Params.featuredImage }} {{ end }}
{{ if not $bg }} {{ $bg = site.Params.defaultBackgroundImage }} {{ end }}
```

> Ce code applique une logique de fallback automatique pour afficher une image de fond cohérente, même si certains paramètres ne sont pas renseignés.

---

### 3. CI/CD avec GitHub Actions

J’ai mis en place un pipeline d’**intégration** et de **déploiement continu**.  
À chaque modification poussée sur la branche principale, le workflow :

- construit automatiquement le site statique avec Hugo,
- génère un artefact optimisé,
- déploie le site sur GitHub Pages.
  Ce processus garantit que le site est toujours à jour en production, sans intervention manuelle.

_Extrait du Fichier :_ `.github/workflows/deploy.yml`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
          extended: true
      - name: Build site
        run: hugo --minify
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

- **Déclencheurs**

  - Se lance automatiquement à chaque `push` sur la branche principale.
  - Peut aussi être déclenché manuellement via GitHub Actions.

- **Permissions**

  - Donne accès en lecture au contenu du repo.
  - Autorise l’écriture sur GitHub Pages.
  - Gère les jetons d’authentification nécessaires.

- **Job 1 : Build**

  - **Checkout** : récupère le code source du dépôt (et les sous-modules si présents).
  - **Setup Hugo** : installe la dernière version de Hugo (extended).
  - **Build site** : génère le site statique avec `hugo --minify`.
  - **Upload artifact** : envoie le dossier `public` (site généré) comme artefact pour le déploiement.

- **Job 2 : Deploy**
  - Dépend du job **Build** (ne démarre qu’après la construction).
  - **Déploiement sur GitHub Pages** : publie l’artefact sur GitHub Pages.
  - Fournit l’URL du site déployé dans l’environnement `github-pages` (`${{ steps.deployment.outputs.page_url }}` est une variable fournie par fournie par l'action officielle `actions/deploy-pages@v4`).

![screenshot github](github-page-action.png "Page 'Actions' sur Github illustrant le déclenchement et l'execution du workflow de façon automatique (on: push)")

---

## Diagramme du pipeline de déploiement

{{< mermaid >}}
flowchart TD
A[Développement local] --> B[Git Commit & Push]
B --> C[GitHub Actions - Build Hugo]
C --> D[Publication sur GitHub Pages]
D --> E[Site en ligne prêt]
{{< /mermaid >}}

> Ce diagramme illustre le pipeline complet du projet, depuis le développement local jusqu’à la mise en ligne automatisée via GitHub Actions.

---

## Conclusion personnelle

En développant ce portfolio j'ai surtout apprécié la fluidité du workflow Hugo (une seule commande pour un aperçu instantané et gestion intuitive des pages) ainsi que la propreté apportée par les shortcodes/partials.
La configuration via **submodule** pour le thème, ainsi que la mise en place de **GitHub Actions**, m'ont permis de renforcer mes compétences **DevOps** et d'**automatiser** un processus qui reste simple à maintenir.
J'ai aussi eu l'occasion de naviguer la **documentation officielle** et les "issues" postées sur Github pour résoudre des problèmes, un travail d'investigation classique en développement logiciel.
