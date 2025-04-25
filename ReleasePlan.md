
# Plan Détaillé – 15 Semaines

## Table des Matières
1. [Phase 1 : Infrastructure Intensive, Tests & Fondations (Semaine 1)](#phase-1--infrastructure-intensive-tests--fondations-semaine-1)  
2. [Phase 2 : Services Client & Fondation Profil (Semaines 2–5)](#phase-2--services-client--fondation-profil-semaines-2–5)  
3. [Phase 3 : Paiement, Stock & Admin (Semaines 6–11)](#phase-3--paiement-stock--admin-semaines-6–11)  
4. [Phase 4 : Permissions & Support (Semaines 12–14)](#phase-4--permissions--support-semaines-12–14)  
5. [Phase 5 : Finalisation & Polissage (Semaine 15)](#phase-5--finalisation--polissage-semaine-15)  
6. [Backlog & User Stories](#backlog--user-stories)  
   - [Microservices & BD](#microservices--bd)  
   - [II – Epics & Liens](#ii--epics--liens)  
   - [Détails des User Stories](#détails-des-user-stories)  

---

## Phase 1 : Infrastructure Intensive, Tests & Fondations (Semaine 1)
<details>
<summary><strong>Semaine 1 (27 Avril – 3 Mai)</strong></summary>

**Objectif**  
Atteindre un déploiement K8s stable et testé des 5 services existants ET de l’infrastructure (auth-db, product-db, elasticsearch, consul, kafka, zookeeper) sur K8s local, avec pipeline CI/CD Jenkins.

**Activités :**
- **Setup Kubernetes & Infra Core**  
  - Installer/configurer K8s local  
  - Manifests K8s pour auth-db, product-db, elasticsearch, consul, kafka, zookeeper  
- **Tests Services Existants** (Jest, Supertest)  
  - Auth: [US-1.1](#us-1-1), [US-1.3](#us-1-3), `/validate`, `/me`  
  - Product: CRUD, Kafka, Image, Search ([US-6.1](#us-6-1))  
  - Gateway, etc.  
- **Déploiement des Services**  
  - Deployment, Service, ConfigMap, Secret  
  - Variables d’env. via DNS K8s  
- **Stabilisation**  
  - readiness/liveness probes, requests/limits, debug pods  
- **Affinage Jenkins**  
  - Tests → build & push images → `kubectl apply` → `rollout status`

**Livrable**  
Cluster K8s stable + 5 micro-services + infra core, pipeline automatisé.

</details>

---

## Phase 2 : Services Client & Fondation Profil (Semaines 2–5)

<details>
<summary><strong>Semaine 2 (4 Mai – 10 Mai) : Cart Service – Logique Core</strong></summary>

- Création **cart-service**, déploiement **cart-db** (Postgres)  
- APIs CRUD panier ([US-3.1](#us-3-1) à [US-3.4](#us-3-4)), invités/connectés ([US-3.6](#us-3-6))  
- Tests Jest/Supertest, pipeline, intégration Gateway  

**Livrable** : Cart Service testé et déployé.
</details>

<details>
<summary><strong>Semaine 3 (11 Mai – 17 Mai) : User Service & Création Admin</strong></summary>

- Création **user-service**, déploiement **user-db**, schéma Prisma  
- Auth Service: événement Kafka `UserRegistered`, POST `/admin/users/create-admin` ([US-1.12](#us-1-12))  
- User Service: consommer `UserRegistered`, GET/PUT `/users/me` ([US-1.6](#us-1-6)), GET `/admin/users` ([US-1.11](#us-1-11))  
- Tests, pipeline, intégration Gateway  

**Livrable** : User Service et création admins.
</details>

<details>
<summary><strong>Semaine 4 (18 Mai – 24 Mai) : Order Service – Création</strong></summary>

- Création **order-service**, déploiement **order-db**, schéma Prisma (snapshot adresse)  
- POST `/orders` (cartId, userId, shippingAddress) ([US-3.8](#us-3-8), [US-3.9](#us-3-9))  
- Émission `OrderCreated` Kafka, tests, pipeline, integration Gateway  

**Livrable** : Order Service enregistrement commandes.
</details>

<details>
<summary><strong>Semaine 5 (25 Mai – 31 Mai) : Order Service – Récup & Statut</strong></summary>

- GET `/orders` ([US-3.15](#us-3-15)), GET `/orders/{id}` ([US-3.16](#us-3-16))  
- Framework interne de mise à jour statut/historique  
- Tests, pipeline, déploiement  

**Livrable** : Consultation historique/détails.
</details>

---

## Phase 3 : Paiement, Stock & Admin (Semaines 6–11)

<details>
<summary><strong>Semaine 6 (1 Juin – 7 Juin) : Payment Service – Initiation</strong></summary>

- Création **payment-service**, déploiement **payment-db**, schéma Prisma  
- POST `/payments/initiate` (Stripe sandbox) ([US-4.1](#us-4-1)), `PaymentInitiated` Kafka  
- Tests mock Stripe, pipeline, integration Gateway  

**Livrable** : Initiation paiement sandbox.
</details>

<details>
<summary><strong>Semaine 7 (8 Juin – 14 Juin) : Payment Service – Webhook & Intégration</strong></summary>

- POST `/payments/webhook` ([US-4.2](#us-4-2)), events `PaymentSucceeded`/`PaymentFailed`  
- Order Service consomme → mise à jour statut → appel `decrement stock` ([US-5.2](#us-5-2))  
- Tests, pipeline  

**Livrable** : Synchronisation paiement/commande/stock.
</details>

<details>
<summary><strong>Semaine 8 (15 Juin – 21 Juin) : E2E & Vues Admin</strong></summary>

- Tests E2E manuels du flux client  
- Endpoints Admin Order ([US-3.21](#us-3-21), [US-3.23](#us-3-23)), Payment ([US-4.3](#us-4-3), [US-4.4](#us-4-4))  
- Protection rôle ADMIN, tests API  

**Livrable** : Vues Admin opérationnelles.
</details>

<details>
<summary><strong>Semaine 9 (22 Juin – 28 Juin) : Monitoring & Instrumentation</strong></summary>

- Déploiement **Prometheus** & **Grafana**  
- Instrumentation services via **prom-client**, endpoint `/metrics`  
- Dashboards basiques  

**Livrable** : Monitoring en place.
</details>

<details>
<summary><strong>Semaine 10 (29 Juin – 5 Juillet) : Sécurisation Product Service</strong></summary>

- Middleware **adminAuth**, protection endpoints Product ([US-2.1](#us-2-1)–[US-2.7](#us-2-7), [US-5.1](#us-5-1), [US-5.2](#us-5-2))  
- Tests API, pipeline  

**Livrable** : Endpoints Product sécurisés.
</details>

<details>
<summary><strong>Semaine 11 (6 Juillet – 12 Juillet) : Adresses & Cache Redis</strong></summary>

- Déploiement **Redis**  
- User Service: CRUD adresses ([US-1.7](#us-1-7)–[US-1.10](#us-1-10)), tests  
- Order Service: fetch adresse via API ([US-3.9](#us-3-9)), tests  
- Caching Redis dans services, tests  

**Livrable** : Adresses & cache Redis.
</details>

---

## Phase 4 : Permissions & Support (Semaines 12–14)

<details>
<summary><strong>Semaine 12 (13 Juillet – 19 Juillet) : Rôles & Permissions Dynamiques</strong></summary>

- Auth Service DB Roles/Permissions & APIs CRUD (US-1.13)  
- Middleware **permissionCheck**, sécurisation endpoints admin  
- Tests backend & API, pipeline  

**Livrable** : Permissions fines actives.
</details>

<details>
<summary><strong>Semaine 13 (20 Juillet – 26 Juillet) : Notification & Promotion</strong></summary>

- **Notification Service**: consomme events (UserRegistered, OrderCreated, PaymentSucceeded, OrderShipped, LowStockWarning), envoie emails (Mailtrap) ([US-8.1](#us-8-1), [US-8.2](#us-8-2), [US-8.4](#us-8-4))  
- **Promotion Service**: CRUD codes promo ([US-9.1](#us-9-1)), validation (`/validate/{code}`) ([US-9.8](#us-9-8)), tests  

**Livrable** : Emails & promos basiques.
</details>

<details>
<summary><strong>Semaine 14 (27 Juillet – 2 Août) : Review & Recommendation</strong></summary>

- **Review Service**: POST/GET reviews ([US-?](#))  
- **Recommendation Service**: consomme `OrderItemPurchased`, GET `/recommendations/product/{id}`, tests  

**Livrable** : Reviews & recommandations.
</details>

---

## Phase 5 : Finalisation & Polissage (Semaine 15)
<details>
<summary><strong>Semaine 15 (3 Août – 9 Août)</strong></summary>

- Stub **CMS** & **Analytics** (Kafka events), tests minimaux  
- Tests manuels E2E finaux, correction bugs critiques  
- Documentation READMEs & Swagger/Postman  
- Nettoyage code, vérification pipeline, préparation démo  

**Livrable** : Application complète, documentée, déployable, prête pour présentation.
</details>

---

## Backlog & User Stories

### Microservices & BD

| Service                  | Description                                            |
|--------------------------|--------------------------------------------------------|
| Auth Service             | Auth, sessions, rôles, permissions, tokens             |
| User Service             | Profils, adresses, wishlists                           |
| Product Service          | Catalogue, médias, variantes, stock                    |
| Order Service            | Commandes, états, historique                           |
| Payment Service          | Initiation & validation paiements                      |
| Search Service           | Elasticsearch, events asynchrones                      |
| Analytics Service        | Collecte & analyse données                             |
| Notification Service     | Email, push, SMS                                       |
| Promotion Service        | Coupons, campagnes                                     |
| CMS Service              | Contenu dynamique                                      |
| Recommendation Service   | Suggestions personnalisées                             |
| Review & Ratings Service | Avis & notes produits                                  |
| Cart Service             | Paniers invités & enregistrés                          |

Chaque service a sa propre base PostgreSQL. Data Warehouse analytique (Power BI).

---

### II – Epics & Liens

- **Epic 1 : Gestion Utilisateurs & Auth** → [US-1.1](#us-1-1) → [US-1.14](#us-1-14)  
- **Epic 2 : Gestion Produits** → [US-2.1](#us-2-1) → [US-2.14](#us-2-14)  
- **Epic 3 : Panier & Commandes** → [US-3.1](#us-3-1) → [US-3.26](#us-3-26)  
- **Epic 4 : Paiements** → [US-4.1](#us-4-1) → [US-4.4](#us-4-4)  
- **Epic 5 : Stock** → [US-5.1](#us-5-1) → [US-5.6](#us-5-6)  
- **Epic 6 : Recherche** → [US-6.1](#us-6-1) → [US-6.3](#us-6-3)  
- **Epic 7 : Analytics & Reporting** → [US-7.1](#us-7-1) → [US-7.11](#us-7-11)  
- **Epic 8 : Notifications** → [US-8.1](#us-8-1) → [US-8.5](#us-8-5)  
- **Epic 9 : Promotions** → [US-9.1](#us-9-1) → [US-9.11](#us-9-11)  

---

### Détails des User Stories

<details>
<summary><strong><a name="us-1-1"></a>US-1.1 : Création de compte utilisateur</strong></summary>
En tant que visiteur, je peux créer un compte avec email et mot de passe pour accéder aux fonctionnalités réservées.  
</details>

<details>
<summary><strong><a name="us-1-2"></a>US-1.2 : Inscription via Google</strong></summary>
En tant que visiteur, je peux m’inscrire via mon compte Google pour accélérer l’enregistrement.  
</details>

<details>
<summary><strong><a name="us-1-3"></a>US-1.3 : Connexion utilisateur</strong></summary>
En tant qu’utilisateur, je peux me connecter avec mes identifiants pour accéder à mon espace.  
</details>

<details>
<summary><strong><a name="us-1-4"></a>US-1.4 : Déconnexion</strong></summary>
En tant qu’utilisateur, je peux me déconnecter pour sécuriser ma session.  
</details>

<details>
<summary><strong><a name="us-1-5"></a>US-1.5 : Réinitialisation mot de passe</strong></summary>
En tant qu’utilisateur, je peux réinitialiser mon mot de passe en cas d’oubli.  
</details>

<details>
<summary><strong><a name="us-1-6"></a>US-1.6 : Modification profil</strong></summary>
En tant qu’utilisateur, je peux modifier mes infos perso (nom, email, téléphone).  
</details>

<details>
<summary><strong><a name="us-1-7"></a>US-1.7 : Ajout d’adresses</strong></summary>
En tant qu’utilisateur, je peux ajouter plusieurs adresses de livraison à mon profil.  
</details>

<details>
<summary><strong><a name="us-1-8"></a>US-1.8 : Modification adresses</strong></summary>
En tant qu’utilisateur, je peux modifier mes adresses existantes.  
</details>

<details>
<summary><strong><a name="us-1-9"></a>US-1.9 : Suppression adresses</strong></summary>
En tant qu’utilisateur, je peux supprimer une adresse de mon profil.  
</details>

<details>
<summary><strong><a name="us-1-10"></a>US-1.10 : Adresse par défaut</strong></summary>
En tant qu’utilisateur, je peux définir une adresse par défaut pour la livraison.  
</details>

<details>
<summary><strong><a name="us-1-11"></a>US-1.11 : Liste utilisateurs (Admin)</strong></summary>
En tant que Super Admin, je peux voir la liste de tous les utilisateurs.  
</details>

<details>
<summary><strong><a name="us-1-12"></a>US-1.12 : Ajouter un administrateur</strong></summary>
En tant que Super Admin, je peux créer un admin (email, nom, téléphone, rôle) et lui envoyer un email d’activation.  
</details>

<details>
<summary><strong><a name="us-1-13"></a>US-1.13 : Rôles & permissions</strong></summary>
En tant que Super Admin, je peux créer des rôles personnalisés avec permissions (lire/écrire/supprimer) sur domaines clés.  
</details>

<details>
<summary><strong><a name="us-1-14"></a>US-1.14 : Historique activités admins</strong></summary>
En tant que Super Admin, je peux consulter l’historique des actions des administrateurs.  
</details>

<details>
<summary><strong><a name="us-2-1"></a>US-2.1 : Création catégorie hiérarchique</strong></summary>
En tant qu’admin, je peux créer une catégorie (max 2 niveaux), seules les feuilles contiennent des produits.  
</details>

<details>
<summary><strong><a name="us-2-2"></a>US-2.2 : Modification catégorie</strong></summary>
En tant qu’admin, je peux modifier une catégorie tout en respectant la hiérarchie à 2 niveaux.  
</details>

<details>
<summary><strong><a name="us-2-3"></a>US-2.3 : Création produit</strong></summary>
En tant qu’admin, je peux créer un produit : SKU unique, nom, description, catégories feuilles, variantes (JSON) et quantités.  
</details>

<details>
<summary><strong><a name="us-2-4"></a>US-2.4 : Modification produit</strong></summary>
En tant qu’admin, je peux modifier un produit existant et ses associations catégories.  
</details>

<details>
<summary><strong><a name="us-2-5"></a>US-2.5 : Désactivation produit</strong></summary>
En tant qu’admin, je peux désactiver un produit temporairement sans le supprimer.  
</details>

<details>
<summary><strong><a name="us-2-6"></a>US-2.6 : Création variantes & seuil bas</strong></summary>
En tant qu’admin, je peux créer variantes avec gestion de stock et seuil bas d’alerte.  
</details>

<details>
<summary><strong><a name="us-2-7"></a>US-2.7 : Mise à jour stock variante</strong></summary>
En tant qu’admin, je peux mettre à jour le stock d’une variante.  
</details>

<details>
<summary><strong><a name="us-2-8"></a>US-2.8 : Alerte seuil bas</strong></summary>
En tant qu’admin, je reçois une alerte quand le stock descend sous le seuil défini.  
</details>

<details>
<summary><strong><a name="us-2-9"></a>US-2.9 : Modification variant attributes</strong></summary>
En tant qu’admin, je peux modifier attributs et seuil bas d’une variante existante.  
</details>

<details>
<summary><strong><a name="us-2-10"></a>US-2.10 : Historique mouvements de stock</strong></summary>
En tant qu’admin, je peux consulter l’historique des mouvements de stock par type.  
</details>

<details>
<summary><strong><a name="us-2-11"></a>US-2.11 : Parcourir catégories</strong></summary>
En tant que visiteur/utilisateur, je peux parcourir les catégories hiérarchiques.  
</details>

<details>
<summary><strong><a name="us-2-12"></a>US-2.12 : Voir détails produit</strong></summary>
En tant que visiteur/utilisateur, je peux voir les détails d’un produit et ses variantes.  
</details>

<details>
<summary><strong><a name="us-2-13"></a>US-2.13 : Disponibilité stock</strong></summary>
En tant que visiteur/utilisateur, je peux connaître la disponibilité en stock de chaque variante.  
</details>

<details>
<summary><strong><a name="us-2-14"></a>US-2.14 : Filtrer produits</strong></summary>
En tant que visiteur/utilisateur, je peux filtrer par catégorie, stock et attributs variantes.  
</details>

<details>
<summary><strong><a name="us-3-1"></a>US-3.1 : Ajouter au panier</strong></summary>
En tant que visiteur/utilisateur, je peux ajouter un produit à mon panier.  
</details>

<details>
<summary><strong><a name="us-3-2"></a>US-3.2 : Modifier quantité panier</strong></summary>
En tant que visiteur/utilisateur, je peux modifier la quantité d’un produit dans mon panier.  
</details>

<details>
<summary><strong><a name="us-3-3"></a>US-3.3 : Retirer du panier</strong></summary>
En tant que visiteur/utilisateur, je peux retirer un produit de mon panier.  
</details>

<details>
<summary><strong><a name="us-3-4"></a>US-3.4 : Total panier mis à jour</strong></summary>
En tant que visiteur/utilisateur, je vois le total du panier mis à jour en temps réel.  
</details>

<details>
<summary><strong><a name="us-3-5"></a>US-3.5 : Sauvegarder panier</strong></summary>
En tant qu’utilisateur, je peux sauvegarder mon panier pour y revenir plus tard.  
</details>

<details>
<summary><strong><a name="us-3-6"></a>US-3.6 : Persistance locale panier</strong></summary>
En tant que visiteur, je conserve mon panier même si je quitte le site.  
</details>

<details>
<summary><strong><a name="us-3-7"></a>US-3.7 : Panier multi-appareils</strong></summary>
En tant qu’utilisateur, je retrouve mon panier sur différents appareils.  
</details>

<details>
<summary><strong><a name="us-3-8"></a>US-3.8 : Passage à la caisse</strong></summary>
En tant que visiteur/utilisateur, je peux passer à la caisse avec mon panier.  
</details>

<details>
<summary><strong><a name="us-3-9"></a>US-3.9 : Choix adresse livraison</strong></summary>
En tant que visiteur/utilisateur, je peux choisir une adresse de livraison.  
</details>

<details>
<summary><strong><a name="us-3-10"></a>US-3.10 : Choix méthode expédition</strong></summary>
En tant que visiteur/utilisateur, je peux choisir une méthode d’expédition.  
</details>

<details>
<summary><strong><a name="us-3-11"></a>US-3.11 : Appliquer code promo</strong></summary>
En tant que visiteur/utilisateur, je peux appliquer un code promo à ma commande.  
</details>

<details>
<summary><strong><a name="us-3-12"></a>US-3.12 : Choix mode paiement</strong></summary>
En tant que visiteur/utilisateur, je peux choisir un mode de paiement.  
</details>

<details>
<summary><strong><a name="us-3-13"></a>US-3.13 : Récapitulatif commande</strong></summary>
En tant que visiteur/utilisateur, je vois un récapitulatif avant confirmation.  
</details>

<details>
<summary><strong><a name="us-3-14"></a>US-3.14 : Confirmation email</strong></summary>
En tant que visiteur/utilisateur, je reçois une confirmation par email après commande.  
</details>

<details>
<summary><strong><a name="us-3-15"></a>US-3.15 : Historique commandes</strong></summary>
En tant qu’utilisateur, je peux consulter l’historique de mes commandes.  
</details>

<details>
<summary><strong><a name="us-3-16"></a>US-3.16 : Détails commande</strong></summary>
En tant qu’utilisateur, je peux voir les détails d’une commande.  
</details>

<details>
<summary><strong><a name="us-3-17"></a>US-3.17 : Suivi livraison</strong></summary>
En tant qu’utilisateur, je peux suivre le statut de livraison.  
</details>

<details>
<summary><strong><a name="us-3-18"></a>US-3.18 : Annulation commande</strong></summary>
En tant qu’utilisateur, je peux annuler une commande non expédiée.  
</details>

<details>
<summary><strong><a name="us-3-20"></a>US-3.20 : Notifications statut commande</strong></summary>
En tant qu’utilisateur, je reçois des notifications lors de changements de statut.  
</details>

<details>
<summary><strong><a name="us-3-21"></a>US-3.21 : Voir toutes commandes (Admin)</strong></summary>
En tant qu’admin, je peux voir toutes les commandes passées.  
</details>

<details>
<summary><strong><a name="us-3-22"></a>US-3.22 : Filtrer/rechercher commandes</strong></summary>
En tant qu’admin, je peux filtrer et rechercher des commandes.  
</details>

<details>
<summary><strong><a name="us-3-23"></a>US-3.23 : Détails commande (Admin)</strong></summary>
En tant qu’admin, je peux voir les détails d’une commande.  
</details>

<details>
<summary><strong><a name="us-3-24"></a>US-3.24 : Mettre à jour statut commande</strong></summary>
En tant qu’admin, je peux mettre à jour le statut d’une commande.  
</details>

<details>
<summary><strong><a name="us-3-25"></a>US-3.25 : Ajouter notes internes</strong></summary>
En tant qu’admin, je peux ajouter des notes internes à une commande.  
</details>

<details>
<summary><strong><a name="us-3-26"></a>US-3.26 : Générer factures</strong></summary>
En tant qu’admin, je peux générer des factures pour les commandes.  
</details>

<details>
<summary><strong><a name="us-4-1"></a>US-4.1 : Initier paiement carte</strong></summary>
En tant que visiteur/utilisateur, je peux payer ma commande par carte bancaire.  
</details>

<details>
<summary><strong><a name="us-4-2"></a>US-4.2 : Confirmation paiement</strong></summary>
En tant que visiteur/utilisateur, je reçois une confirmation de paiement.  
</details>

<details>
<summary><strong><a name="us-4-3"></a>US-4.3 : Voir paiements (Admin)</strong></summary>
En tant qu’admin, je peux consulter tous les paiements effectués.  
</details>

<details>
<summary><strong><a name="us-4-4"></a>US-4.4 : Détail paiement (Admin)</strong></summary>
En tant qu’admin, je peux voir le détail d’un paiement.  
</details>

<details>
<summary><strong><a name="us-5-1"></a>US-5.1 : Ajouter stock</strong></summary>
En tant qu’admin, je peux ajouter des articles au stock.  
</details>

<details>
<summary><strong><a name="us-5-2"></a>US-5.2 : Mettre à jour stock</strong></summary>
En tant qu’admin, je peux mettre à jour les quantités en stock.  
</details>

<details>
<summary><strong><a name="us-5-3"></a>US-5.3 : Définir seuil réapprovisionnement</strong></summary>
En tant qu’admin, je peux définir un seuil d’alerte pour le stock.  
</details>

<details>
<summary><strong><a name="us-5-4"></a>US-5.4 : Alerte réapprovisionnement</strong></summary>
En tant qu’admin, je suis alerté quand le stock atteint le seuil.  
</details>

<details>
<summary><strong><a name="us-5-5"></a>US-5.5 : Historique mouvements stock</strong></summary>
En tant qu’admin, je peux voir l’historique des mouvements de stock.  
</details>

<details>
<summary><strong><a name="us-5-6"></a>US-5.6 : Voir disponibilité produit</strong></summary>
En tant que visiteur/utilisateur, je peux voir si un produit est en stock.  
</details>

<details>
<summary><strong><a name="us-6-1"></a>US-6.1 : Recherche par mot-clé</strong></summary>
En tant que visiteur/utilisateur, je peux rechercher des produits par mot-clé.  
</details>

<details>
<summary><strong><a name="us-6-2"></a>US-6.2 : Autocomplétion</strong></summary>
En tant que visiteur/utilisateur, je peux utiliser l’autocomplétion dans la barre de recherche.  
</details>

<details>
<summary><strong><a name="us-6-3"></a>US-6.3 : Suggestions de recherche</strong></summary>
En tant que visiteur/utilisateur, je vois des suggestions basées sur mes requêtes.  
</details>

<details>
<summary><strong><a name="us-7-1"></a>US-7.1 : Suivi comportement utilisateurs</strong></summary>
En tant que Marketing Manager, je peux suivre le comportement des utilisateurs.  
</details>

<details>
<summary><strong><a name="us-7-2"></a>US-7.2 : Taux de conversion & abandon</strong></summary>
En tant que Marketing Manager, je peux suivre conversions et taux d’abandon.  
</details>

<details>
<summary><strong><a name="us-7-3"></a>US-7.3 : Analyse parcours client</strong></summary>
En tant que Marketing Manager, je peux analyser le parcours client.  
</details>

<details>
<summary><strong><a name="us-7-4"></a>US-7.4 : Produits les plus consultés</strong></summary>
En tant que Marketing Manager, je peux voir les produits les plus consultés.  
</details>

<details>
<summary><strong><a name="us-7-5"></a>US-7.5 : Produits ajoutés au panier</strong></summary>
En tant que Marketing Manager, je peux voir les produits les plus ajoutés au panier.  
</details>

<details>
<summary><strong><a name="us-7-6"></a>US-7.6 : Tableau de bord KPI</strong></summary>
En tant que Super Admin, je peux voir un dashboard avec KPIs principaux.  
</details>

<details>
<summary><strong><a name="us-7-7"></a>US-7.7 : Rapports de ventes</strong></summary>
En tant que Marketing Manager, je peux générer des rapports de ventes par période.  
</details>

<details>
<summary><strong><a name="us-7-8"></a>US-7.8 : Rapports performance produits</strong></summary>
En tant que Product Manager, je peux générer des rapports produits.  
</details>

<details>
<summary><strong><a name="us-7-9"></a>US-7.9 : Rapports niveaux de stock</strong></summary>
En tant que Inventory Manager, je peux générer des rapports stock.  
</details>

<details>
<summary><strong><a name="us-7-10"></a>US-7.10 : Rapports financiers</strong></summary>
En tant que Finance Manager, je peux générer des rapports financiers.  
</details>

<details>
<summary><strong><a name="us-7-11"></a>US-7.11 : Export rapports</strong></summary>
En tant que Super Admin, je peux exporter les rapports (PDF, Excel).  
</details>

<details>
<summary><strong><a name="us-8-1"></a>US-8.1 : Email confirmation actions</strong></summary>
En tant qu’utilisateur, je reçois des emails de confirmation pour actions importantes.  
</details>

<details>
<summary><strong><a name="us-8-2"></a>US-8.2 : Notifications statut commande</strong></summary>
En tant qu’utilisateur, je reçois des notifications sur statut commande.  
</details>

<details>
<summary><strong><a name="us-8-3"></a>US-8.3 : Alertes admin actions</strong></summary>
En tant qu’admin, je reçois des alertes pour actions nécessitant mon attention.  
</details>

<details>
<summary><strong><a name="us-8-4"></a>US-8.4 : Alertes stock bas</strong></summary>
En tant qu’admin, je reçois une alerte pour stock sous seuil.  
</details>

<details>
<summary><strong><a name="us-8-5"></a>US-8.5 : Alertes nouvelle commande</strong></summary>
En tant qu’admin, je reçois une alerte pour chaque nouvelle commande.  
</details>

<details>
<summary><strong><a name="us-9-1"></a>US-9.1 : Création codes promo</strong></summary>
En tant qu’admin, je peux créer des codes promotionnels.  
</details>

<details>
<summary><strong><a name="us-9-2"></a>US-9.2 : Conditions promo</strong></summary>
En tant qu’admin, je peux définir des conditions d’utilisation pour les promos.  
</details>

<details>
<summary><strong><a name="us-9-3"></a>US-9.3 : Limite utilisations code</strong></summary>
En tant qu’admin, je peux limiter le nombre d’utilisations d’un code promo.  
</details>

<details>
<summary><strong><a name="us-9-4"></a>US-9.4 : Promotions limitées dans le temps</strong></summary>
En tant qu’admin, je peux créer des promotions avec date de début/fin.  
</details>

<details>
<summary><strong><a name="us-9-6"></a>US-9.6 : Réductions catégories</strong></summary>
En tant qu’admin, je peux configurer des réductions sur certaines catégories.  
</details>

<details>
<summary><strong><a name="us-9-7"></a>US-9.7 : Offres spéciales (2+1)</strong></summary>
En tant qu’admin, je peux configurer des offres spéciales (2+1 gratuit, etc.).  
</details>

<details>
<summary><strong><a name="us-9-8"></a>US-9.8 : Appliquer code promo (client)</strong></summary>
En tant que visiteur/utilisateur, je peux appliquer un code promo à mon panier.  
</details>

<details>
<summary><strong><a name="us-9-9"></a>US-9.9 : Promotions sur page d’accueil</strong></summary>
En tant que visiteur/utilisateur, je peux voir les promotions actuelles sur la home.  
</details>

<details>
<summary><strong><a name="us-9-10"></a>US-9.10 : Voir produits en promo</strong></summary>
En tant que visiteur/utilisateur, je peux voir les produits en promotion.  
</details>

<details>
<summary><strong><a name="us-9-11"></a>US-9.11 : Codes promo personnalisés</strong></summary>
En tant qu’utilisateur, je reçois des codes promo personnalisés.  
</details>
