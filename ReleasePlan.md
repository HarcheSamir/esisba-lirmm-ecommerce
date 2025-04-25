# Plan Détaillé - 15 Semaines

## Phase 1 : Infrastructure Intensive, Tests & Fondations (Semaine 1)

### Semaine 1 (27 Avril – 3 Mai) : Déploiement K8s (Infra Core & Services Existants), Tests Rigoureux & Affinage Pipeline Jenkins

**Objectif :**  
Atteindre un déploiement K8s stable et testé des 5 services existants ET de TOUS les composants d'infrastructure requis (auth-db, product-db, elasticsearch, consul, kafka, zookeeper) sur K8s local, géré par un pipeline CI/CD Jenkins affiné ciblant K8s.

**Activités Détaillées :**
- **Setup Kubernetes & Déploiement Infra Core**  
  - Installer/Configurer K8s local.  
  - Créer/Déployer les manifests K8s pour PostgreSQL (auth-db, product-db), Elasticsearch (cluster), Consul (serveur), Kafka (brokers) et ZooKeeper.  
  - Vérifier accessibilité et état Ready de chaque Pod.
- **Tests des Services Existants** (Jest, Supertest)  
  - Auth Service : logique principale ([US-1.1](#US-1.1), [US-1.3](#US-1.3), `/validate`, `/me`).  
  - Product Service : CRUD Product/Category/Variant/Stock, production Kafka, image upload/retrieve, search ([US-6.1](#US-6.1)).  
  - Gateway : discovery, proxy, rewrite.
- **Déploiement K8s des Services**  
  - Manifests : Deployment, Service, ConfigMap, Secret pour chaque service.  
  - Variables d’env. pointant vers l’infra via DNS K8s.
- **Stabilisation**  
  - Readiness/Liveness probes.  
  - Resource requests/limits.  
  - Debug & validation des Pods Ready.
- **Affinage Pipeline Jenkins**  
  - Jenkinsfile :  
    1. Checkout  
    2. Tests Jest/Supertest (fail on error)  
    3. Build/Tag/Push images  
    4. `kubectl apply` + `rollout status`

**Livrable :**  
Cluster K8s stable avec infra core + 5 microservices initiaux. Pipeline Jenkins automatisé.

---

## Phase 2 : Services du Flux Client Principal & Fondation Profil Utilisateur (Semaines 2-5)

### Semaine 2 (4 Mai – 10 Mai) : Cart Service – Implémentation Logique Core

**Objectif :**  
Implémenter les fonctionnalités principales du Cart Service.

**Activités :**
- Créer `cart-service`, setup cart-db (Postgres en K8s).  
- APIs CRUD du panier :  
  - [US-3.1](#US-3.1) Ajouter produit  
  - [US-3.2](#US-3.2) Modifier quantité  
  - [US-3.3](#US-3.3) Retirer produit  
  - [US-3.4](#US-3.4) Voir total en temps réel  
- Gestion utilisateurs invités/connectés ([US-3.6](#US-3.6)).  
- Tests Jest/Supertest.  
- Intégration au pipeline & déploiement K8s.  
- Intégration routes au Gateway.

**Livrable :**  
Cart Service fonctionnel en K8s, tests verts.

---

### Semaine 3 (11 Mai – 17 Mai) : Fondation User Service & Création Admin User (Auth Service)

**Objectif :**  
Établir les bases du User Service et permettre la création d’admins via l’Auth Service.

**Activités :**
- Créer `user-service`, setup user-db (Postgres en K8s), schéma Prisma profil basique.  
- Auth Service :  
  - Production event Kafka `UserRegistered`.  
  - POST `/admin/users/create-admin` ([US-1.12](#US-1.12)).  
  - Gestion rôle/JWT.  
- User Service :  
  - Consommer `UserRegistered`.  
  - GET/PUT `/users/me` ([US-1.6](#US-1.6)).  
  - GET `/admin/users` ([US-1.11](#US-1.11)).  
- Tests Jest/Supertest.  
- Déploiement via Jenkins & intégration Gateway.

**Livrable :**  
User Service déployé, gestion profil basique et création d’admins validée.

---

### Semaine 4 (18 Mai – 24 Mai) : Order Service – Schéma & Création (Adresse Basique)

**Objectif :**  
Mettre en place l’Order Service pour la création de commandes avec adresse en snapshot.

**Activités :**
- Créer `order-service`, setup order-db (Postgres en K8s), schéma Prisma (snapshot adresse).  
- POST `/orders` acceptant `cartId`, `userId`, `shippingAddress` ([US-3.8](#US-3.8), partiel [US-3.9](#US-3.9)).  
- Fetch panier (Cart API), création Order/Items, event Kafka `OrderCreated`.  
- Tests Jest/Supertest, Jenkins, Gateway.

**Livrable :**  
Order Service crée commandes, stocke adresse, produit event.

---

### Semaine 5 (25 Mai – 31 Mai) : Order Service – Statut & Récupération

**Objectif :**  
Récupération commandes client et framework de mise à jour de statut.

**Activités :**
- GET `/orders` ([US-3.15](#US-3.15)), GET `/orders/{orderId}` ([US-3.16](#US-3.16)).  
- Mécanisme interne de statut & historique.  
- Tests Jest/Supertest, déploiement Jenkins.

**Livrable :**  
API commandes consultable par client, framework statut établi.

---

## Phase 3 : Intégration Paiement, Stock & Fondations Admin (Semaines 6-11)

### Semaine 6 (1 Juin – 7 Juin) : Payment Service – Setup, Schéma & Initiation

**Objectif :**  
Mettre en place le Payment Service et initier paiement sandbox.

**Activités :**
- Créer `payment-service`, payment-db (Postgres en K8s), schéma Prisma.  
- POST `/payments/initiate` ([US-4.1](#US-4.1) partiel).  
- Intégration Stripe Sandbox, stockage PENDING, event Kafka `PaymentInitiated`.  
- Tests Jest/Supertest (mock Stripe), Jenkins, Gateway.

**Livrable :**  
Payment Service initie paiements sandbox.

---

### Semaine 7 (8 Juin – 14 Juin) : Payment Service – Webhook & Intégration

**Objectif :**  
Gérer statuts paiement, mise à jour commande & stock.

**Activités :**
- POST `/payments/webhook` ([US-4.2](#US-4.2) partiel).  
- MàJ record paiement, events `PaymentSucceeded`/`PaymentFailed`.  
- Consumer Order Service : update statut, appel API décrémentation stock ([US-5.2](#US-5.2)).  
- Implémentation API interne Product Service.  
- Tests Jest/Supertest, déploiement.

**Livrable :**  
Paiement traité, commande & stock mis à jour.

---

### Semaine 8 (15 Juin – 21 Juin) : Test E2E & Vues Admin Basiques

**Objectif :**  
Tester le flux principal E2E et implémenter vues Admin commandes et paiements.

**Activités :**
- Tests manuels E2E sur K8s, doc résultats.  
- Order Service Admin :  
  - GET `/admin/orders` ([US-3.21](#US-3.21)), GET `/admin/orders/{id}` ([US-3.23](#US-3.23)).  
- Payment Service Admin :  
  - GET `/admin/payments` ([US-4.3](#US-4.3)), GET `/admin/payments/{id}` ([US-4.4](#US-4.4)).  
- Protection rôles ADMIN, tests intégration, déploiement.

**Livrable :**  
Flux E2E testé, endpoints Admin fonctionnels & testés.

---

### Semaine 9 (22 Juin – 28 Juin) : Monitoring & Instrumentation

**Objectif :**  
Déployer Prometheus/Grafana et instrumenter services.

**Activités :**
- Manifests K8s pour Prometheus & Grafana.  
- Ajout de `prom-client` sur services, endpoint `/metrics`.  
- Configuration scrape & dashboards basiques.

**Livrable :**  
Monitoring en place, services metrics exposés.

---

### Semaine 10 (29 Juin – 5 Juillet) : Sécurisation Endpoints Product/Stock

**Objectif :**  
Appliquer protection ADMIN aux endpoints existants du Product Service.

**Activités :**
- `adminAuthMiddleware` sur US-2.1→US-2.7, US-2.9, US-2.10, US-5.1, US-5.2, US-5.5.  
- Mise à jour tests Jest/Supertest.  
- Déploiement.

**Livrable :**  
Product Service sécurisé, tests validés.

---

### Semaine 11 (6 Juillet – 12 Juillet) : Adresses User Service & Cache Redis

**Objectif :**  
Implémenter CRUD adresses, intégration commande et caching Redis.

**Activités :**
- Redis K8s.  
- User Service : endpoints adresses ([US-1.7](#US-1.7), [US-1.8](#US-1.8), [US-1.9](#US-1.9), [US-1.10](#US-1.10)).  
- Order Service : POST `/orders` avec `shippingAddressId`, fetch User Service ([US-3.9](#US-3.9)).  
- Intégration Redis dans services core, tests cache.  
- Déploiement.

**Livrable :**  
Gestion adresses & caching Redis opérationnels.

---

## Phase 4 : Permissions Fines & Services de Support (Semaines 12-14)

### Semaine 12 (13 Juillet – 19 Juillet) : Rôles & Permissions Dynamiques

**Objectif :**  
Implémenter gestion rôles/permissions et vérifications fines.

**Activités :**
- Auth Service : schéma DB (Roles, Permissions, RolePermissions), CRUD APIs, assignations.  
- JWT inclut permissions.  
- Middleware `permissionCheck(requiredPermission)` sur tous les endpoints admin.  
- Tests Unit & API pour Auth + intégration.  
- Déploiement.

**User Story : [US-1.13](#US-1.13)**

**Livrable :**  
Permissions fines en place, tests validés.

---

### Semaine 13 (20 Juillet – 26 Juillet) : Notification & Promotion Services

**Objectif :**  
Implémenter Notification Service et Promotion Service basique.

**Activités :**
- **Notification Service** :  
  - Consommer events (`UserRegistered`, `OrderCreated`, `PaymentSucceeded`, `OrderShipped` [US-3.20], `LowStockWarning` [US-8.4]).  
  - SendGrid/Mailtrap, envoi emails ([US-8.1](#US-8.1), [US-8.2](#US-8.2)).  
- **Product Service** : low stock check ([US-5.3](#US-5.3)), event `LowStockWarning` ([US-2.8](#US-2.8)).  
- **Promotion Service** :  
  - POST `/admin/promotions` ([US-9.1](#US-9.1)), GET `/promotions/validate/{code}`.  
  - Order Service: appliquer réduction ([US-3.11](#US-3.11), [US-9.8](#US-9.8)).  
- Tests Jest/Supertest, déploiement.

**Livrable :**  
Notifications & promos basiques fonctionnels et testés.

---

### Semaine 14 (27 Juillet – 2 Août) : Review & Recommendation Services

**Objectif :**  
Implémenter Review & Recommendation Services basiques.

**Activités :**
- **Review Service** : POST `/reviews`, GET `/products/{id}/reviews`, tests Jest/Supertest.  
- **Recommendation Service** : consommer `OrderItemPurchased`, stocker co-achat, GET `/recommendations/product/{id}`, tests Jest.

**Livrable :**  
Reviews et recommandations basiques opérationnels.

---

## Phase 5 : Finalisation & Polissage (Semaine 15)

### Semaine 15 (3 Août – 9 Août) : Stub CMS, Analytics, Tests E2E, Documentation & Buffer

**Objectif :**  
Ajouter stubs CMS/Analytics, tests finaux, doc & présentation.

**Activités :**
- Stub CMS : GET `/content/{key}`.  
- Stub Analytics : consommer Kafka (`US-7.1`→`US-7.5` partiel).  
- Product Service: event `ProductViewed`.  
- Tests Manuels E2E finaux, doc et correctifs critiques.  
- Génération README & docs API.  
- Préparation présentation/démo, pipeline Jenkins vérifié.

**Livrable :**  
Microservices documentés, testés E2E, K8s local prêt, permissions fines, monitoring, flows core, documentation complète.

---

# Backlog et User Stories - Application E-commerce Mono-commerçant

## Microservices & Bases de données

| Service                     | Description                                                                                              |
|-----------------------------|----------------------------------------------------------------------------------------------------------|
| Auth Service                | Authentification, gestion des sessions, rôles, permissions, tokens, etc.                                  |
| User Service                | Gestion des utilisateurs : profils, adresses, wishlists                                                   |
| Product Service             | Catalogue produits, médias, catégories, variants, stock                                                   |
| Order Service               | Gestion des commandes, états, historique                                                                   |
| Payment Service             | Traitement et validation des paiements                                                                     |
| Search Service              | Recherche (Elasticsearch), communication asynchrone avec Produits                                          |
| Analytics Service           | Collecte et analyse des données utilisateur                                                               |
| Notification Service        | Envoi notifications (email, push, SMS)                                                                     |
| Promotion Service           | Gestion promotions, coupons, campagnes                                                                     |
| CMS Service                 | Contenu dynamique (bannières, textes, styles)                                                              |
| Recommendation Service      | Suggestions personnalisées                                                                                 |
| Review & Ratings Service    | Avis et notes produits                                                                                     |
| Cart Service                | Gestion des paniers utilisateurs (invités et enregistrés)                                                 |

PostgreSQL dédiée par service pour isolation et scalabilité  
Data Warehouse analytique avec visualisation Power BI

---

## Epics et User Stories

### Epic 1 : Gestion des Utilisateurs et Authentification

<a name="US-1.1"></a>#### US-1.1 :  
En tant que visiteur, je peux créer un compte avec email/mot de passe pour accéder aux fonctionnalités réservées.

<a name="US-1.2"></a>#### US-1.2 :  
En tant que visiteur, je peux m’inscrire via mon compte Google.

<a name="US-1.3"></a>#### US-1.3 :  
En tant qu’utilisateur enregistré, je peux me connecter à mon compte.

<a name="US-1.4"></a>#### US-1.4 :  
En tant qu’utilisateur enregistré, je peux me déconnecter.

<a name="US-1.5"></a>#### US-1.5 :  
En tant qu’utilisateur enregistré, je peux réinitialiser mon mot de passe.

<a name="US-1.6"></a>#### US-1.6 :  
En tant qu’utilisateur enregistré, je peux modifier mes informations personnelles.

#### 1.2 Gestion des Adresses

<a name="US-1.7"></a>#### US-1.7 :  
En tant qu’utilisateur enregistré, je peux ajouter des adresses de livraison.

<a name="US-1.8"></a>#### US-1.8 :  
En tant qu’utilisateur enregistré, je peux modifier mes adresses.

<a name="US-1.9"></a>#### US-1.9 :  
En tant qu’utilisateur enregistré, je peux supprimer mes adresses.

<a name="US-1.10"></a>#### US-1.10 :  
En tant qu’utilisateur enregistré, je peux définir une adresse par défaut.

#### 1.3 Administration des Utilisateurs

<a name="US-1.11"></a>#### US-1.11 :  
En tant que Super Admin, je peux voir la liste de tous les utilisateurs.

<a name="US-1.12"></a>#### US-1.12 :  
En tant que Super Admin, je peux ajouter un administrateur (email, nom, téléphone, rôle).

<a name="US-1.13"></a>#### US-1.13 :  
En tant que Super Admin, je peux créer des rôles personnalisés avec permissions fines.

<a name="US-1.14"></a>#### US-1.14 :  
En tant que Super Admin, je peux accéder à l’historique des activités des admins.

---

### Epic 2 : Gestion des Produits

<a name="US-2.1"></a>#### US-2.1 :  
En tant qu’admin, je peux créer une catégorie hiérarchique (2 niveaux max).

<a name="US-2.2"></a>#### US-2.2 :  
En tant qu’admin, je peux modifier une catégorie existante.

<a name="US-2.3"></a>#### US-2.3 :  
En tant qu’admin, je peux créer un produit (SKU, nom, desc, catégories, variantes).

<a name="US-2.4"></a>#### US-2.4 :  
En tant qu’admin, je peux modifier un produit existant.

<a name="US-2.5"></a>#### US-2.5 :  
En tant qu’admin, je peux désactiver un produit temporairement.

#### 2.2 Variantes & Stock

<a name="US-2.6"></a>#### US-2.6 :  
En tant qu’admin, je peux créer des variantes avec seuil bas.

<a name="US-2.7"></a>#### US-2.7 :  
En tant qu’admin, je peux mettre à jour le stock d’une variante.

<a name="US-2.8"></a>#### US-2.8 :  
En tant qu’admin, je reçois une alerte quand stock < seuil.

<a name="US-2.9"></a>#### US-2.9 :  
En tant qu’admin, je peux modifier les attributs et seuil des variantes.

<a name="US-2.10"></a>#### US-2.10 :  
En tant qu’admin, je peux consulter l’historique des mouvements de stock.

#### 2.3 Consultation

<a name="US-2.11"></a>#### US-2.11 :  
Parcourir les catégories.

<a name="US-2.12"></a>#### US-2.12 :  
Voir les détails d’un produit.

<a name="US-2.13"></a>#### US-2.13 :  
Voir disponibilité en stock.

<a name="US-2.14"></a>#### US-2.14 :  
Filtrer par catégorie, disponibilité, attributs.

---

### Epic 3 : Gestion du Panier et des Commandes

#### 3.1 Panier

<a name="US-3.1"></a>#### US-3.1 :  
Ajouter un produit au panier.

<a name="US-3.2"></a>#### US-3.2 :  
Modifier la quantité.

<a name="US-3.3"></a>#### US-3.3 :  
Retirer un produit.

<a name="US-3.4"></a>#### US-3.4 :  
Voir total mis à jour.

<a name="US-3.5"></a>#### US-3.5 :  
Sauvegarder le panier.

<a name="US-3.6"></a>#### US-3.6 :  
Persistence locale pour visiteurs.

<a name="US-3.7"></a>#### US-3.7 :  
Récupérer panier multi-appareils.

#### 3.2 Processus de Commande

<a name="US-3.8"></a>#### US-3.8 :  
Passer à la caisse.

<a name="US-3.9"></a>#### US-3.9 :  
Choisir une adresse de livraison.

<a name="US-3.10"></a>#### US-3.10 :  
Choisir une méthode d’expédition.

<a name="US-3.11"></a>#### US-3.11 :  
Appliquer un code promo.

<a name="US-3.12"></a>#### US-3.12 :  
Choisir un mode de paiement.

<a name="US-3.13"></a>#### US-3.13 :  
Voir récapitulatif avant confirmation.

<a name="US-3.14"></a>#### US-3.14 :  
Confirmation par email après commande.

#### 3.3 Suivi des Commandes

<a name="US-3.15"></a>#### US-3.15 :  
Consulter l’historique des commandes.

<a name="US-3.16"></a>#### US-3.16 :  
Voir détails d’une commande.

<a name="US-3.17"></a>#### US-3.17 :  
Suivre le statut de livraison.

<a name="US-3.18"></a>#### US-3.18 :  
Annuler une commande non expédiée.

<a name="US-3.20"></a>#### US-3.20 :  
Recevoir notifications statut.

#### 3.4 Admin Commandes

<a name="US-3.21"></a>#### US-3.21 :  
Voir toutes les commandes.

<a name="US-3.22"></a>#### US-3.22 :  
Filtrer/rechercher des commandes.

<a name="US-3.23"></a>#### US-3.23 :  
Voir détails d’une commande.

<a name="US-3.24"></a>#### US-3.24 :  
Mettre à jour statut.

<a name="US-3.25"></a>#### US-3.25 :  
Ajouter notes internes.

<a name="US-3.26"></a>#### US-3.26 :  
Générer factures.

---

### Epic 4 : Gestion des Paiements

#### 4.1 Traitement

<a name="US-4.1"></a>#### US-4.1 :  
Payer par carte bancaire.

<a name="US-4.2"></a>#### US-4.2 :  
Recevoir confirmation de paiement.

#### 4.2 Admin Paiements

<a name="US-4.3"></a>#### US-4.3 :  
Voir tous les paiements en tant qu’admin.

<a name="US-4.4"></a>#### US-4.4 :  
Voir détails d’un paiement.

---

### Epic 5 : Gestion du Stock

<a name="US-5.1"></a>#### US-5.1 :  
Ajouter des articles au stock.

<a name="US-5.2"></a>#### US-5.2 :  
Mettre à jour les quantités en stock.

<a name="US-5.3"></a>#### US-5.3 :  
Définir seuil d’alerte.

<a name="US-5.4"></a>#### US-5.4 :  
Recevoir alertes stock bas.

<a name="US-5.5"></a>#### US-5.5 :  
Voir historique mouvements de stock.

<a name="US-5.6"></a>#### US-5.6 :  
Voir disponibilité produit.

---

### Epic 6 : Moteur de Recherche

<a name="US-6.1"></a>#### US-6.1 :  
Rechercher produits par mot-clé.

<a name="US-6.2"></a>#### US-6.2 :  
Auto-complétion.

<a name="US-6.3"></a>#### US-6.3 :  
Suggestions basées sur requêtes.

---

### Epic 7 : Analytics et Reporting

#### 7.1 Collecte

<a name="US-7.1"></a>#### US-7.1 :  
Suivre comportement utilisateur.

<a name="US-7.2"></a>#### US-7.2 :  
Suivre conversions & abandons.

<a name="US-7.3"></a>#### US-7.3 :  
Analyser parcours client.

<a name="US-7.4"></a>#### US-7.4 :  
Voir produits les plus consultés.

<a name="US-7.5"></a>#### US-7.5 :  
Voir produits ajoutés au panier.

#### 7.2 Dashboards & Rapports

<a name="US-7.6"></a>#### US-7.6 :  
Tableau de bord KPIs.

<a name="US-7.7"></a>#### US-7.7 :  
Rapports de ventes.

<a name="US-7.8"></a>#### US-7.8 :  
Rapports performance produits.

<a name="US-7.9"></a>#### US-7.9 :  
Rapports niveaux de stock.

<a name="US-7.10"></a>#### US-7.10 :  
Rapports financiers.

<a name="US-7.11"></a>#### US-7.11 :  
Exporter rapports (PDF, Excel).

---

### Epic 8 : Système de Notifications

#### 8.1 Utilisateurs

<a name="US-8.1"></a>#### US-8.1 :  
Recevoir emails de confirmation.

<a name="US-8.2"></a>#### US-8.2 :  
Recevoir notifications statut commandes.

#### 8.2 Administrateurs

<a name="US-8.3"></a>#### US-8.3 :  
Alertes pour actions critiques.

<a name="US-8.4"></a>#### US-8.4 :  
Alertes produits sous seuil.

<a name="US-8.5"></a>#### US-8.5 :  
Alertes nouvelles commandes.

---

### Epic 9 : Système de Promotions

#### 9.1 Gestion

<a name="US-9.1"></a>#### US-9.1 :  
Créer codes promo.

<a name="US-9.2"></a>#### US-9.2 :  
Définir conditions d’utilisation.

<a name="US-9.3"></a>#### US-9.3 :  
Limiter nombre d’utilisations.

<a name="US-9.4"></a>#### US-9.4 :  
Promotions limitées dans le temps.

<a name="US-9.6"></a>#### US-9.6 :  
Réductions sur catégories.

<a name="US-9.7"></a>#### US-9.7 :  
Offres spéciales (2+1 gratuit).

#### 9.2 Utilisation

<a name="US-9.8"></a>#### US-9.8 :  
Appliquer un code promo au panier.

<a name="US-9.9"></a>#### US-9.9 :  
Voir promos sur la page d’accueil.

<a name="US-9.10"></a>#### US-9.10 :  
Voir produits en promotion.

<a name="US-9.11"></a>#### US-9.11 :  
Recevoir codes personnalisés.

