# Portail de la Jeunesse Istiqlalienne — بوابة الشبيبة الاستقلالية

Portail officiel bilingue (français / arabe) de la Jeunesse Istiqlalienne, organisation de jeunesse du Parti de l'Istiqlal.

> **Maquette de démonstration.** Les contenus, chiffres et documents présentés sont fournis à titre d'illustration et ne constituent pas une publication officielle de l'organisation.

## Aperçu en ligne

La maquette est publiée via GitHub Pages : **https://<utilisateur>.github.io/<depot>/**

*(remplacez `<utilisateur>` et `<depot>` après le premier déploiement)*

## Contenu du dépôt

| Chemin | Description |
|---|---|
| `index.html` | **Maquette v2** — fichier autonome, publié par GitHub Pages |
| `src/v1_demo_portail_officiel.html` | Maquette v1 d'origine, conservée comme référence de charte |
| `src/v1_organigramme_interactif_arabe.html` | Organigramme autonome d'origine (source des 35 fiches) |
| `docs/ARCHITECTURE_PHASE0.md` | Architecture technique et modèle de données du portail Laravel |

## La maquette v2

Fichier HTML **autonome** : CSS et JavaScript intégrés, aucune dépendance externe hormis la police Tajawal (Google Fonts) pour l'arabe. Il s'ouvre directement dans un navigateur, sans serveur ni installation.

### Ce qu'elle apporte par rapport à la v1

- **Organigramme intégré** à l'onglet Gouvernance, restylé à la charte : arbre hiérarchique cliquable à 6 niveaux, grille des 35 membres du Bureau exécutif, panneau de détail affichant la mission officielle.
- **Navigation réorganisée** en 4 rubriques de contenu (L'organisation, Participer, Ressources, Contact) et une zone d'actions distincte (Rechercher, Espace membre, Adhérer).
- **Bilinguisme complet** : un bouton bascule toute l'interface et applique `dir="rtl"` sur `<html>`. La mise en page utilise les propriétés logiques CSS, sans feuille de style miroir.
- **Accessibilité** : navigation clavier intégrale, focus visible, `aria-live` sur les zones dynamiques, cibles tactiles de 44 px minimum, `prefers-reduced-motion` respecté.
- **URLs partageables** : chaque page a son adresse (`#/organisation/gouvernance`) et le bouton retour du navigateur fonctionne.

### Modifier les contenus

Les textes sont regroupés dans des objets JavaScript en tête du fichier, avant tout balisage. Le responsable communication peut les éditer sans toucher au code :

| Objet | Rôle |
|---|---|
| `MENU` | Arborescence des menus (libellés FR et AR) |
| `UI` | Libellés d'interface, messages d'erreur et d'état |
| `ORG_LEVELS` | Les 6 niveaux de l'organigramme, leur description et leur effectif |
| `POLES` | Les 10 pôles thématiques et leur libellé bilingue |
| `MEMBERS` | Les 35 membres : nom, pôle, mission officielle |

**Deux règles à respecter :**

1. Le champ `mission` reproduit le texte officiel arabe. Il ne doit pas être reformulé.
2. Les effectifs inconnus s'affichent `—` ou « en cours d'annonce ». Aucun chiffre n'est inventé.

### Les pôles thématiques

Les 34 membres du Bureau exécutif portent tous le même intitulé officiel (« عضو المكتب التنفيذي »), ce qui rendait la liste illisible. Dix pôles ont été **déduits du texte de leurs missions** pour permettre le filtrage :

| Pôle | Membres |
|---|---|
| Formation & jeunesse | 6 |
| Social & solidarité | 5 |
| Organisation & finances | 4 |
| Territoires & sections | 4 |
| Communication & numérique | 4 |
| Économie & développement | 4 |
| Doctrine & analyse politique | 3 |
| Institutions & plaidoyer | 2 |
| International & MRE | 2 |
| Direction | 1 |

Ce regroupement est une proposition : **il doit être validé ou corrigé par le Bureau exécutif**. Il se modifie dans l'objet `POLES` et le champ `pole` de chaque membre.

Un champ `kw` accompagne chaque membre. Il n'est jamais affiché : il contient des mots-clés français qui rendent la recherche possible en français alors que les missions officielles sont rédigées en arabe. Il peut être enrichi librement.

## Suite du projet

La maquette sert de référence visuelle et fonctionnelle au développement du portail réel, prévu en **Laravel 12** (Filament pour le back-office, Livewire pour le front public). L'architecture technique et le modèle de données figurent dans `docs/ARCHITECTURE_PHASE0.md`.

## Licence

Contenus institutionnels et éléments d'identité visuelle : propriété de la Jeunesse Istiqlalienne. Reproduction soumise à autorisation.
