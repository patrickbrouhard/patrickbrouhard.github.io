---

title: "Mes Voyages"
date: 2025-10-15
summary: "Application Symfony full-stack pour enregistrer, trier et présenter des visites de voyages (CRUD, upload d'images, tri/filtre, envoi de contact, tests et migrations)."
tags: ["symfony", "php", "doctrine", "twig", "vich-uploader", "mailer", "tests", "migrations"]
cover: "images/visites/003-chicago-immeubles-6628146d1ace2614016903.jpg"
draft: false
------------

## Présentation du projet

Dans le cadre de ma formation, je suis en train de développer **MesVoyages**, une application web réalisée avec le framework **Symfony** destinée à stocker, organiser et présenter des visites personnelles (ville, pays, date, note, avis, images). 
Le dépôt contient l'ensemble du code applicatif (controllers, entités, formulaires, templates Twig), les migrations Doctrine, des fixtures et des tests unitaires/ fonctionnels.

Ce projet se focalise essentiellement sur l'application des bonnes pratiques de développement backend (ORM, validation, upload d'images, envoi d'e-mails, tests, CI/CD).

{{< button href="https://github.com/patrickbrouhard/mesvoyages" target="_blank" rel="noopener" >}}
<svg xmlns="http://www.w3.org/2000/svg" 
     class="w-5 h-5 mr-2 inline-block" 
     fill="currentColor" viewBox="0 0 16 16">
<path d="M8 0C3.58 0 0 3.58 0 8a8 
           8 0 0 0 5.47 7.59c.4.07.55-.17.55-.38 
           0-.19-.01-.82-.01-1.49-2 
           .37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13
           -.28-.15-.68-.52-.01-.53.63-.01 1.08.58 
           1.23.82.72 1.21 1.87.87 
           2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95
           0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12
           0 0 .67-.21 2.2.82a7.65 7.65 0 0 1 2-.27c.68 
           0 1.36.09 2 .27 1.53-1.04 2.2-.82 
           2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 
           1.27.82 2.15 0 3.07-1.87 3.75-3.65 
           3.95.29.25.54.73.54 1.48 
           0 1.07-.01 1.93-.01 2.2 
           0 .21.15.46.55.38A8 8 0 0 0 16 
           8c0-4.42-3.58-8-8-8z"/>
</svg>
Voir sur GitHub
{{< /button >}}


## Objectifs et valeur ajoutée

* Fournir une application complète de gestion de contenus (CRUD) centrée sur des visites de voyage.
* Mettre en œuvre des pratiques robustes : validations côté serveur, gestion d'assets (VichUploader), tests automatisés, et processus de migration.
* Présenter un projet prêt à être déployé (migrations, configuration, workflows GitHub présents) pour convaincre des recruteurs ou pour servir de base à un portfolio.

## Fonctionnalités principales

* CRUD complet des entités *Visite* (création, lecture, mise à jour, suppression).
* Upload et gestion d'images liées aux visites (VichUploader).
* Tri et filtrage des visites (par ville, pays, date, note).
* Page d'accueil et listing des voyages (interface publique).
* Interface d'administration sécurisée (authentification).
* Formulaire de contact et envoi d'emails via Symfony Mailer.
* Migrations Doctrine et fixtures pour l'initialisation de la BDD.
* Tests unitaires et fonctionnels (PHPUnit, bundles de test).
* Fichiers de configuration pour CI/CD et qualité (GitHub Actions, sonar...).

## Technologies utilisées

* PHP >= 8.1
* Symfony (6.x)
* Doctrine ORM + Migrations
* Twig (templates)
* VichUploaderBundle (upload d'images)
* Symfony Mailer
* PHPUnit / bundles de test
* Bootstrap (front minimal)
* GitHub Actions (workflows présent dans `.github/workflows`)
* MySQL / Doctrine DBAL pour la persistance

## Compétences démontrées

**Techniques**

* Conception d'entités via Doctrine, relations et migrations. (une entité dans ce contexte est une classe qui représente une table de la BDD, 1 instance = 1 ligne)
* Mise en place de formulaires Symfony et validation (constraints).
* Intégration d'un gestionnaire d'uploads (VichUploader) et stockage d'assets.
* Tests automatisés (unit / fonctionnel) et fixtures pour données de test.
* Configuration et automatisation (migrations, CI/CD).

**Transversales**

* Respect des bonnes pratiques (séparation controller/service/repository).
* Structuration d'un projet pour production (migrations, configuration).
* Documentation et préparation pour déploiement continu.
* Communication claire du code (noms explicites, commit history structurée).

## Extraits de code représentatifs

1. **Entité `Visite` — mapping/validation (src/Entity/Visite.php)**
   Cet extrait montre l'utilisation de Doctrine attributes, VichUploader et des contraintes de validation.

```php
#[ORM\Column(length: 50)]
private ?string $ville = null;

#[ORM\Column(type: Types::DATE_MUTABLE, nullable: true)]
#[Assert\LessThanOrEqual("now")]
private ?\DateTimeInterface $datecreation = null;

#[ORM\Column(nullable: true)]
#[Assert\Range(min: 0, max: 20)]
private ?int $note = null;
```

*Explication :* définition claire des colonnes, contraintes métiers directement dans l'entité pour garantir l'intégrité des données.
- `#[Assert\LessThanOrEqual("now")]` sur `$datecreation`  
  → règle métier : une date de création ne peut pas être dans le futur.  
- `#[Assert\Range(min: 0, max: 20)]` sur `$note`  
  → règle métier : la note doit être comprise entre 0 et 20 (comme une note scolaire).  
- `#[ORM\Column(length: 50)]` sur `$ville`  
  → contrainte technique (longueur max en BDD), mais qui peut aussi refléter une règle métier si, par exemple, on décide qu’un nom de ville ne dépassera jamais 50 caractères.

2. **Contrôleur d'accueil — récupération des dernières visites (src/Controller/AccueilController.php)**

```php
#[Route('/', name: 'accueil')]
public function index(): Response {
    $visites = $this->repository->findAllLasted(2);
    return $this->render("pages/accueil.html.twig", [
        'visites' => $visites
    ]);
}
```

*Explication :* logique concise : déléguer les requêtes SQL complexes au repository et garder le controller léger (single responsibility).
Le contrôleur ici n'a pas besoin de savoir comment la requête est construite, il appelle simplement findAllLasted(2).

3. **Envoi d'email via Symfony Mailer (src/Controller/ContactController.php)**

```php
$email = (new Email())
    ->from($contact->getEmail())
    ->to('contact@mesvoyages.com')
    ->subject('Message du site de voyages')
    ->html($this->renderView('pages/_email.html.twig', ['contact' => $contact]));
$mailer->send($email);
```

*Explication :* intégration du mailer pour un workflow de contact utilisateur, avec rendu HTML depuis un template Twig.

<!-- ## Modélisation des données du projet MesVoyages

{{< mermaid >}}
erDiagram
    VISITE {
        int id PK
        string ville
        string pays
        date datecreation
        int note
        string avis
        string image
        datetime dateenregistrement
    }

    ENVIRONNEMENT {
        int id PK
        string nom
    }

    UTILISATEUR {
        int id PK
        string email
        string password
        string roles
    }

    VISITE ||--o{ ENVIRONNEMENT : "a lieu dans"
    UTILISATEUR ||--o{ VISITE : "crée"
{{< /mermaid >}} -->



## Conclusion personnelle

Ce projet m'a permis d'apprendre à utiliser le framework Symfony (entités, forms, services) et d'aborder des préoccupations réelles de production : validation, gestion d'assets, migrations et tests automatisés. 
J'ai également progressé sur la structuration du code pour la maintenance et sur la préparation d'un pipeline de déploiement.

---
