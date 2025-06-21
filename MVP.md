
LIRMM-ECommerce : Documentation Technique d'une Plateforme E-Commerce à base de Microservices

Auteur : Samir

Directeur de Projet : [Nom du Directeur]

Date : 24 Mai 2024

Version : 1.1 (Révision Détaillée)

1. Synthèse du Projet

Le projet LIRMM-ECommerce constitue une plateforme backend d'e-commerce, conçue sur une architecture microservices pour garantir une haute disponibilité, une scalabilité horizontale et une maintenabilité optimale. Le système a été décomposé en services autonomes suivant les principes du Domain-Driven Design (DDD), où chaque microservice encapsule une responsabilité métier spécifique, de l'authentification des utilisateurs à la gestion des commandes.

L'épine dorsale de la communication inter-services repose sur un modèle événementiel (Event-Driven), orchestré par Apache Kafka, qui assure un découplage temporel et une résilience accrue entre les composants. La pile technologique a été soigneusement sélectionnée pour sa maturité et sa pertinence dans l'écosystème cloud-native : Node.js/Express.js pour la logique applicative, PostgreSQL pour la persistance transactionnelle, Elasticsearch pour les capacités de recherche avancée, Redis pour la gestion de cache, et Consul pour la découverte de services.

L'ensemble de l'écosystème est conteneurisé avec Docker et orchestré par Kubernetes, simulant un environnement de production. Une pipeline CI/CD automatisée avec Jenkins assure la construction, le test et le déploiement continus de l'application, incarnant les meilleures pratiques DevOps.

2. Architecture du Système
2.1. Principes Architecturaux Fondamentaux

Décomposition par Domaine Métier : Chaque service (Auth, Product, Order, etc.) est le "propriétaire" exclusif d'un sous-domaine de l'application. Cette séparation garantit une haute cohésion interne et un couplage externe faible.

Point d'Entrée Unique (API Gateway) : La Passerelle API agit comme une façade, simplifiant l'interaction pour les clients et encapsulant la complexité du réseau interne de microservices.

Asynchronisme et Découplage (Event-Driven) : L'utilisation de Kafka comme bus de messages est un choix stratégique. Il permet aux services de communiquer sans dépendance directe et immédiate. Un service producteur émet un événement sans attendre la réaction des consommateurs. Cela permet une scalabilité indépendante des consommateurs et une résilience aux pannes.

Base de Données par Service : Ce principe fondamental garantit l'autonomie complète de chaque service. Il n'y a pas de base de données partagée. La cohérence des données entre les services est assurée par la synchronisation événementielle.

Infrastructure as Code (IaC) : L'entièreté de l'infrastructure est définie de manière déclarative via des fichiers docker-compose.yml et des manifestes Kubernetes, ce qui rend l'environnement entièrement reproductible et versionnable.

2.2. Diagramme d'Architecture de Haut Niveau
Generated mermaid
graph TD
    subgraph "Monde Extérieur"
        User[<fa:fa-user> Utilisateur / Client]
    end

    subgraph "Cluster Kubernetes"
        AGW[<fa:fa-server> Passerelle API]

        subgraph "Services Applicatifs"
            Auth[<fa:fa-id-card> Service d'Authentification]
            Product[<fa:fa-box> Service de Produits]
            ImageSvc[<fa:fa-image> Service d'Images]
            Cart[<fa:fa-shopping-cart> Service de Panier]
            Order[<fa:fa-receipt> Service de Commandes]
            Search[<fa:fa-search> Service de Recherche]
        end

        subgraph "Infrastructure & Persistance"
            Consul[<fa:fa-compass> Consul (Service Discovery)]
            Kafka[<fa:fa-random> Bus d'Événements Kafka]
            Elastic[<fa:fa-search-plus> Cluster Elasticsearch]
            AuthDB[(<fa:fa-database> DB Auth<br>Postgres)]
            ProductDB[(<fa:fa-database> DB Produit<br>Postgres)]
            OrderDB[(<fa:fa-database> DB Commande<br>Postgres)]
            Redis[(<fa:fa-database> Cache Redis)]
        end
    end

    User --> AGW

    AGW -- "Route /auth" --> Auth
    AGW -- "Route /products" --> Product
    AGW -- "Route /images" --> ImageSvc
    AGW -- "Route /search" --> Search
    AGW -- "Route /carts" --> Cart
    AGW -- "Route /orders" --> Order

    Auth -- "S'enregistre auprès de" --> Consul
    Product -- "S'enregistre auprès de" --> Consul
    Search -- "S'enregistre auprès de" --> Consul
    Cart -- "S'enregistre auprès de" --> Consul
    Order -- "S'enregistre auprès de" --> Consul
    ImageSvc -- "S'enregistre auprès de" --> Consul
    AGW -- "Découvre les services via" --> Consul

    Auth -- "Persiste dans" --> AuthDB
    Product -- "Persiste dans" --> ProductDB
    Order -- "Persiste dans" --> OrderDB
    Cart -- "Utilise le cache" --> Redis
    Search -- "Indexe dans" --> Elastic

    Auth -- "Produit 'auth_events'" --> Kafka
    Product -- "Produit 'product_events'" --> Kafka

    Kafka -- "Consomme 'auth_events'" --> Order
    Kafka -- "Consomme 'product_events'" --> Order
    Kafka -- "Consomme 'product_events'" --> Search

3. Analyse Détaillée des Microservices
3.1. Passerelle API (API Gateway)

Mission : Point d'entrée unique et façade pour le système. Elle est responsable du routage des requêtes externes vers les services internes appropriés, en se basant sur la découverte de services dynamique.

Pile Technologique : Node.js, Express.js, http-proxy-middleware, Consul.

Logique Clé : Utilise une fonction router asynchrone qui interroge Consul à chaque requête pour obtenir l'URL saine et à jour du service de destination.

Tableau des Routes de Proxy :

Préfixe de Route	Service de Destination	Réécriture de Chemin	Description
/auth	auth-service	Aucune	Route les requêtes d'authentification et de gestion des utilisateurs.
/products	product-service	Aucune	Route les requêtes vers le catalogue de produits.
/images	image-service	^/images -> /	Route les requêtes d'images, en enlevant le préfixe /images.
/search	search-service	Aucune	Route les requêtes de recherche vers le service dédié.
/carts	cart-service	Aucune	Route les requêtes de gestion de panier.
/orders	order-service	Aucune	Route les requêtes de gestion de commande.
3.2. Service d'Authentification (Auth Service)

Mission : Gère l'identité, l'authentification, les rôles et les permissions (RBAC). Il est la source de vérité pour les utilisateurs et émet/valide les JWT.

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL, Bcrypt, JWT, KafkaJS.

Interactions Événementielles : Agit en tant que producteur sur le topic Kafka auth_events.

Modèle de Données (auth_db via Prisma):

Generated prisma
model User { id, name, email, password, isActive, profileImage, roleId, role }
model Role { id, name, description, users, permissions }
model Permission { id, name, description, roles }
model RolePermission { roleId, permissionId, role, permission }
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Prisma
IGNORE_WHEN_COPYING_END

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description	Permissions Requises
POST	/register	Crée un nouvel utilisateur.	Publique
POST	/login	Authentifie un utilisateur et retourne un JWT.	Publique
GET	/me	Retourne les détails de l'utilisateur authentifié.	Token valide
POST	/validate	(Interne) Valide un JWT pour d'autres services.	Aucune (interne au cluster)
GET	/users	Liste tous les utilisateurs.	read:user
GET, PUT	/users/:id	Lit ou met à jour un utilisateur spécifique.	read:user, write:user
DELETE	/users/:id	Désactive un utilisateur.	delete:user
POST	/users/:id/activate	Réactive un utilisateur.	write:user
GET, POST	/roles	Liste ou crée des rôles.	read:role, write:role
GET, PUT, DELETE	/roles/:id	CRUD sur un rôle spécifique.	read:role, write:role
GET	/permissions	Liste toutes les permissions système.	read:role
3.3. Service de Produits (Product Service)

Mission : Source de vérité pour l'ensemble du catalogue : produits, catégories, variantes, images et niveaux de stock.

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL, KafkaJS.

Interactions Événementielles : Producteur sur le topic product_events.

Modèle de Données (product_db via Prisma):

Generated prisma
model Category { id, name, slug, parentId, children, ... }
model Product { id, sku, name, description, variants, categories, images, ... }
model ProductImage { id, imageUrl, isPrimary, order, ... }
model Variant { id, attributes, price, stockQuantity, ... }
model StockMovement { id, changeQuantity, type, reason, ... }
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Prisma
IGNORE_WHEN_COPYING_END

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description	Permissions Requises
GET	/	Liste les produits avec filtres et pagination.	Publique
POST	/	Crée un nouveau produit.	create:product
POST	/bulk	Crée plusieurs produits en une seule requête.	create:product
GET	/id/:id	Récupère un produit par son ID.	Publique
GET	/sku/:sku	Récupère un produit par son SKU.	Publique
PUT, DELETE	/:id	Met à jour ou supprime un produit.	update:product, delete:product
GET, POST	/categories	Liste ou crée des catégories.	Publique / create:category
PUT, DELETE	/categories/:id	Met à jour ou supprime une catégorie.	update:category, delete:category
POST	/:productId/variants	Ajoute une variante à un produit.	update:product
POST	/stock/adjust/:variantId	Ajuste le stock d'une variante.	adjust:stock
3.4. Service d'Images (Image Service)

Mission : Service simple et spécialisé pour le traitement des uploads d'images et leur diffusion statique.

Pile Technologique : Node.js, Express.js, Multer.

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description
POST	/upload	Accepte une image (imageFile) et retourne son URL publique.
GET	/images/:filename	Sert le fichier image statique correspondant.
3.5. Service de Recherche (Search Service)

Mission : Fournir une expérience de recherche textuelle rapide, pertinente et à facettes sur le catalogue de produits.

Pile Technologique : Node.js, Express.js, Client Elasticsearch, KafkaJS.

Interactions Événementielles : Agit en tant que consommateur du topic product_events.

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description
GET	/products	Cherche des produits. Accepte les paramètres q, category, minPrice, maxPrice, page, limit.
3.6. Service de Panier (Cart Service)

Mission : Gérer les paniers d'achat avec une haute performance.

Pile Technologique : Node.js, Express.js, ioredis.

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description
POST	/	Crée ou récupère un panier.
GET	/:cartId	Récupère le contenu d'un panier.
POST	/:cartId/items	Ajoute un article au panier.
PUT	/:cartId/items/:itemId	Met à jour la quantité d'un article.
DELETE	/:cartId/items/:itemId	Supprime un article du panier.
DELETE	/:cartId	Supprime entièrement un panier.
POST	/associate	Associe un panier de "guest" à un userId.
3.7. Service de Commandes (Order Service)

Mission : Orchestrer la création et la gestion des commandes.

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL, KafkaJS.

Interactions Événementielles : Consommateur des topics product_events et auth_events pour maintenir ses tables dénormalisées.

Tableau des Points d'API (Endpoints) :

Méthode HTTP	Route	Description	Permissions Requises
POST	/	Crée une nouvelle commande (guest ou authentifié).	Publique (authentification optionnelle)
GET	/my-orders	Récupère les commandes de l'utilisateur authentifié.	read:my-orders
GET	/:id	Récupère les détails d'une commande.	Publique
GET	/	Liste toutes les commandes.	Admin (adminOnlyMiddleware)
PUT	/:id/status	Met à jour le statut d'une commande.	Admin (adminOnlyMiddleware)
POST	/guest-lookup	Permet à un invité de retrouver sa commande.	Publique
4. Infrastructure et Concepts Transverses
4.1. Communication Événementielle via Apache Kafka

Kafka est la colonne vertébrale asynchrone du système. Il permet une architecture réactive et résiliente en découplant les services. Plutôt que de s'appeler directement, les services communiquent en produisant et consommant des événements.

Rôle Stratégique :

Découplage : Les services producteurs n'ont pas connaissance des services consommateurs, et vice-versa.

Persistance des Événements : Kafka stocke les messages, permettant aux consommateurs hors-ligne de rattraper leur retard.

Scalabilité : Les topics peuvent être partitionnés, permettant à plusieurs instances d'un même service consommateur de se répartir la charge.

Tableau des Flux de Données Événementiels :

Topic	Producteurs	Consommateurs	Événements Clés	Objectif Stratégique
product_events	product-service	search-service, order-service	PRODUCT_CREATED, _UPDATED, _DELETED	Synchroniser l'état du catalogue de produits à travers le système pour la recherche et l'archivage des commandes.
auth_events	auth-service	order-service	USER_CREATED, _UPDATED, _DELETED	Maintenir une copie dénormalisée des informations utilisateur pour enrichir l'historique des commandes.
4.2. Découverte de Services via HashiCorp Consul

Rôle : Annuaire centralisé et dynamique des services.

Mécanisme : Au démarrage, chaque microservice s'enregistre auprès de Consul, fournissant son nom, son adresse IP (obtenue via la Downward API de Kubernetes), son port et un endpoint de health check.

4.3. Indexation et Recherche via Elasticsearch

Rôle : Moteur de recherche textuelle.

Mécanisme : Le search-service maintient un index (products) dont le mapping est optimisé pour la recherche. Il consomme les product_events pour rester synchronisé en quasi-temps réel.

4.4. Gestion de Cache et de Données Éphémères via Redis

Rôle : Base de données en mémoire pour des accès à très faible latence.

Mécanisme : Utilisé par le cart-service. Les paniers sont stockés sous forme de chaînes JSON avec un TTL pour purger automatiquement les données abandonnées.

4.5. Persistance des Données via PostgreSQL

Rôle : Système de gestion de base de données relationnelle pour les données nécessitant une intégrité transactionnelle.

Mécanisme : Trois instances de base de données distinctes (auth_db, product_db, order_db) sont déployées, concrétisant le pattern "Database per Service".

5. Conteneurisation, Orchestration et Intégration Continue (CI/CD)
5.1. Conteneurisation avec Docker

Chaque service est encapsulé dans une image Docker légère et reproductible (node:18-alpine). Le fichier docker-compose.yml orchestre l'ensemble pour le développement local, avec des volumes et un "watch mode" pour le hot-reloading.

5.2. Orchestration avec Kubernetes (Kind)

L'application est déployée sur un cluster Kubernetes local (via Kind), utilisant des manifestes déclaratifs.

Tableau des Objets Kubernetes Clés :

Objet Kubernetes	Rôle dans l'Architecture
Deployment	Gère le cycle de vie des Pods, assure le nombre de répliques et pilote les mises à jour.
Service	Fournit une abstraction réseau stable (ex: auth-service-svc) pour accéder aux Pods.
Init Container	Crucial : Exécute les migrations de base de données (prisma db push & seed) avant que le conteneur applicatif ne démarre.
Volume	Fournit un stockage persistant (simulé avec emptyDir, à remplacer par des PersistentVolumeClaims en production).
5.3. Pipeline CI/CD avec Jenkins

Le Jenkinsfile automatise entièrement le cycle de vie de l'application.

Étapes Séquentielles de la Pipeline :

Checkout : Récupération du code source.

Build Custom Docker Images : Construction des images Docker pour chaque service, taguées de manière unique.

Setup Kind Cluster : Création d'un cluster Kubernetes éphémère et pré-chargement de toutes les images.

Deploy Application to Kind : Rendu du manifeste Kubernetes avec les bons tags et déploiement. La pipeline attend ensuite activement la fin du déploiement.

Integration/E2E Tests : Étape réservée pour les futurs tests d'intégration.

6. Diagrammes de Flux de Données Clés
6.1. Flux d'Enregistrement Utilisateur et Synchronisation de Données
Generated mermaid
sequenceDiagram
    participant C as Client
    participant AGW as Passerelle API
    participant Auth as Service Auth
    participant Kafka as Bus Kafka
    participant Order as Service Commandes

    C->>+AGW: POST /auth/register (données utilisateur)
    AGW->>+Auth: POST /register
    Auth->>Auth: Hachage mdp, création User dans DB
    Auth-->>-AGW: 201 Created (JWT)
    AGW-->>-C: 201 Created (JWT)

    Auth->>Kafka: Produit événement (type: USER_CREATED)
    Note over Kafka: Topic: auth_events

    Kafka-->>Order: Consomme événement
    Order->>Order: Crée/Met à jour DenormalizedUser dans sa DB
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
6.2. Flux de Création de Produit et d'Indexation pour la Recherche
Generated mermaid
sequenceDiagram
    participant Admin as Admin
    participant AGW as Passerelle API
    participant Product as Service Produit
    participant Kafka as Bus Kafka
    participant Search as Service Recherche

    Admin->>+AGW: POST /products (données produit)
    AGW->>+Product: POST /
    Product->>Product: Crée le produit dans la DB (transaction)
    Product-->>-AGW: 201 Created
    AGW-->>-Admin: 201 Created

    Product->>Kafka: Produit événement (type: PRODUCT_CREATED, payload complet)
    Note over Kafka: Topic: product_events

    Kafka-->>Search: Consomme événement
    Search->>Search: Transforme payload et indexe dans Elasticsearch
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
7. Conclusion et Perspectives d'Évolution

Ce projet démontre avec succès la mise en œuvre d'une architecture microservices complexe et fonctionnelle. La séparation des domaines, l'utilisation d'un bus d'événements et l'automatisation via CI/CD constituent une base solide pour une application d'e-commerce moderne.

Pistes d'Amélioration et Perspectives :

Sécurité Avancée : Intégration d'un système de gestion de secrets (ex: HashiCorp Vault).

Observabilité : Déploiement d'une pile de monitoring (Prometheus, Grafana) et de logging centralisé (ELK Stack).

Tests de Contrat : Mise en place de tests de contrat (ex: avec Pact) pour garantir la non-régression des API inter-services.

Patterns de Résilience : Implémentation du pattern Circuit Breaker pour les appels synchrones.

Déploiement Cloud : Transition de Kind vers un service Kubernetes managé (EKS, GKE, AKS).
