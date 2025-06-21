Absolument. Voici une documentation complète et approfondie, rédigée en français avec un ton académique et professionnel, qui met l'accent sur la complexité et les justifications architecturales, en particulier sur les aspects événementiels, comme demandé.

LIRMM-ECommerce : Documentation Technique d'une Plateforme E-Commerce à base de Microservices

Auteur : Samir

Directeur de Projet : [Nom du Directeur]

Date : 24 Mai 2024

Version : 1.1 (Révision Détaillée)

Table des Matières

Synthèse du Projet

Architecture du Système

Principes Architecturaux Fondamentaux

Diagramme d'Architecture de Haut Niveau

Analyse Détaillée des Microservices

Passerelle API (API Gateway)

Service d'Authentification (Auth Service)

Service de Produits (Product Service)

Service d'Images (Image Service)

Service de Recherche (Search Service)

Service de Panier (Cart Service)

Service de Commandes (Order Service)

Infrastructure et Concepts Transverses

Communication Événementielle via Apache Kafka

Découverte de Services via HashiCorp Consul

Indexation et Recherche via Elasticsearch

Gestion de Cache et de Données Éphémères via Redis

Persistance des Données via PostgreSQL

Conteneurisation, Orchestration et Intégration Continue (CI/CD)

Conteneurisation avec Docker

Orchestration avec Kubernetes (Kind)

Pipeline CI/CD avec Jenkins

Diagrammes de Flux de Données Clés

Flux d'Enregistrement Utilisateur et Synchronisation de Données

Flux de Création de Produit et d'Indexation pour la Recherche

Conclusion et Perspectives d'Évolution

1. Synthèse du Projet

Le projet LIRMM-ECommerce constitue une plateforme backend d'e-commerce, conçue sur une architecture microservices pour garantir une haute disponibilité, une scalabilité horizontale et une maintenabilité optimale. Le système a été décomposé en services autonomes suivant les principes du Domain-Driven Design (DDD), où chaque microservice encapsule une responsabilité métier spécifique, de l'authentification des utilisateurs à la gestion des commandes.

L'épine dorsale de la communication inter-services repose sur un modèle événementiel (Event-Driven), orchestré par Apache Kafka, qui assure un découplage temporel et une résilience accrue entre les composants. La pile technologique a été soigneusement sélectionnée pour sa maturité et sa pertinence dans l'écosystème cloud-native : Node.js/Express.js pour la logique applicative, PostgreSQL pour la persistance transactionnelle, Elasticsearch pour les capacités de recherche avancée, Redis pour la gestion de cache, et Consul pour la découverte de services.

L'ensemble de l'écosystème est conteneurisé avec Docker et orchestré par Kubernetes, simulant un environnement de production. Une pipeline CI/CD automatisée avec Jenkins assure la construction, le test et le déploiement continus de l'application, incarnant les meilleures pratiques DevOps.

2. Architecture du Système
2.1. Principes Architecturaux Fondamentaux

Décomposition par Domaine Métier : Chaque service (Auth, Product, Order, etc.) est le "propriétaire" exclusif d'un sous-domaine de l'application. Cette séparation garantit une haute cohésion interne et un couplage externe faible.

Point d'Entrée Unique (API Gateway) : La Passerelle API agit comme une façade, simplifiant l'interaction pour les clients et encapsulant la complexité du réseau interne de microservices.

Asynchronisme et Découplage (Event-Driven) : L'utilisation de Kafka comme bus de messages est un choix stratégique. Il permet aux services de communiquer sans dépendance directe et immédiate. Un service producteur émet un événement sans attendre la réaction des consommateurs. Cela permet une scalabilité indépendante des consommateurs et une résilience aux pannes (un consommateur hors-ligne peut rattraper les événements manqués à sa reconnexion).

Base de Données par Service : Ce principe fondamental garantit l'autonomie complète de chaque service. Il n'y a pas de base de données partagée. Chaque service choisit la technologie de persistance la plus adaptée et peut faire évoluer son schéma sans impacter les autres. La cohérence des données entre les services est assurée par la synchronisation événementielle.

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

Pile Technologique : Node.js, Express.js, http-proxy-middleware.

Logique Clé : Utilise une fonction router asynchrone qui interroge Consul à chaque requête pour obtenir l'URL saine et à jour du service de destination. Une gestion d'erreur robuste (onError) capture les échecs de proxy (ex: service indisponible) et retourne des codes HTTP pertinents (503 Service Unavailable, 502 Bad Gateway).

Points d'API (Routes de Proxy) :

/auth/* -> auth-service

/products/*, /categories/*, /stock/* -> product-service

/images/* -> image-service (avec réécriture de chemin)

/search/* -> search-service

/carts/* -> cart-service

/orders/* -> order-service

3.2. Service d'Authentification (Auth Service)

Mission : Gère l'identité, l'authentification (inscription, connexion), les rôles et les permissions (RBAC). Il est la source de vérité pour les utilisateurs et émet/valide les JSON Web Tokens (JWT).

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL, Bcrypt, JWT.

Modèle de Données (auth_db):

Generated prisma
model User { id, name, email, password, isActive, profileImage, roleId, role }
model Role { id, name, description, users, permissions }
model Permission { id, name, description, roles }
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Prisma
IGNORE_WHEN_COPYING_END

Interactions Événementielles : Agit en tant que producteur sur le topic Kafka auth_events. Il publie des événements (USER_CREATED, USER_UPDATED, USER_DELETED) à chaque modification d'un utilisateur, permettant à d'autres services de maintenir une copie locale synchronisée des données utilisateur.

Points d'API :

POST /register, POST /login: Inscription et connexion.

POST /validate: Endpoint critique et interne, utilisé par les autres services pour valider un JWT.

GET /me: Détails de l'utilisateur authentifié.

CRUD complet sur /roles et /users pour l'administration.

3.3. Service de Produits (Product Service)

Mission : Source de vérité pour l'ensemble du catalogue : produits, catégories, variantes, images et niveaux de stock. Toute modification du catalogue originate ici.

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL.

Modèle de Données (product_db): Schéma relationnel complexe modélisant les produits, leurs variantes (avec attributs JSON), leurs catégories (hiérarchiques), leurs images (avec un ordre de priorité) et les mouvements de stock détaillés.

Interactions Événementielles : Producteur principal sur le topic product_events. Chaque création, mise à jour ou suppression d'un produit, d'une variante ou d'une catégorie déclenche l'émission d'un événement riche contenant l'état complet de l'objet métier.

Points d'API :

CRUD sur /, /id/:id, /sku/:sku.

POST /bulk: Endpoint optimisé pour la création en masse de produits.

CRUD sur /categories.

Gestion des associations produit-catégorie et produit-image.

Routes imbriquées (/:productId/variants) pour la gestion des variantes.

POST /stock/adjust/:variantId: Endpoint transactionnel pour la modification des stocks.

3.4. Service d'Images (Image Service)

Mission : Service simple et spécialisé pour le traitement des uploads d'images et leur diffusion statique, isolant ainsi les opérations I/O lourdes du reste de l'application.

Pile Technologique : Node.js, Express.js, Multer.

Stratégie de Persistance : Utilise un volume persistant Docker/Kubernetes (/app/uploads) pour stocker les fichiers de manière découplée du cycle de vie du conteneur.

3.5. Service de Recherche (Search Service)

Mission : Fournir une expérience de recherche textuelle rapide, pertinente et à facettes sur le catalogue de produits.

Pile Technologique : Node.js, Express.js, Client Elasticsearch, KafkaJS.

Logique Clé : Agit en tant que consommateur du topic product_events. À la réception d'un événement, il transforme la charge utile (payload) en un document JSON plat et structuré, puis l'indexe ou le met à jour dans Elasticsearch. Cette transformation inclut l'aplatissement des attributs de variantes pour faciliter le filtrage (variant_attributes_flat).

Points d'API :

GET /products: Endpoint de recherche flexible acceptant une query (q), des filtres (catégorie, prix, etc.) et la pagination.

3.6. Service de Panier (Cart Service)

Mission : Gérer les paniers d'achat. Conçu pour une performance maximale et des données de nature éphémère.

Pile Technologique : Node.js, Express.js, ioredis.

Stratégie de Persistance : Utilise Redis comme base de données clé-valeur. Chaque panier est un string JSON stocké sous une clé cart:<uuid>. Un TTL (Time-To-Live) est systématiquement appliqué pour purger automatiquement les paniers abandonnés.

Points d'API :

POST /associate: Endpoint clé qui associe un panier de "guest" à un userId après une connexion réussie.

3.7. Service de Commandes (Order Service)

Mission : Orchestrer la création et la gestion des commandes. Il est conçu pour être un "coffre-fort" historique.

Pile Technologique : Node.js, Express.js, Prisma, PostgreSQL, KafkaJS.

Logique Clé (Dénormalisation) : Pour garantir l'intégrité historique des commandes, ce service est un consommateur des topics product_events et auth_events. Il maintient des copies locales et simplifiées (denormalized_products, denormalized_users) des produits et utilisateurs. Ainsi, si un produit est supprimé du catalogue principal, les détails de ce produit sur les commandes passées restent intacts. Lors de la création d'une commande, il effectue un appel synchrone au product-service pour ajuster le stock de manière transactionnelle.

4. Infrastructure et Concepts Transverses
4.1. Communication Événementielle via Apache Kafka

Kafka est la colonne vertébrale asynchrone du système, permettant une architecture réactive et résiliente.

Rôle Stratégique :

Découplage : Les services producteurs n'ont pas connaissance des services consommateurs, et vice-versa.

Persistance des Événements : Kafka stocke les messages pour une durée configurable, permettant aux consommateurs hors-ligne de rattraper leur retard.

Scalabilité : Les topics peuvent être partitionnés, permettant à plusieurs instances d'un même service consommateur de se répartir la charge de travail.

Flux de Données Événementiels Détaillés :

Topic	Producteurs	Consommateurs	Événements Clés	Objectif
product_events	product-service	search-service, order-service	PRODUCT_CREATED, _UPDATED, _DELETED	Synchroniser l'état du catalogue de produits à travers le système.
auth_events	auth-service	order-service	USER_CREATED, _UPDATED, _DELETED	Maintenir une copie dénormalisée des informations utilisateur pour l'historique des commandes.

Diagramme de Flux Événementiel :

Generated mermaid
graph TD
    subgraph "Services Producteurs"
        ProductSvc[<fa:fa-box> Product Service]
        AuthSvc[<fa:fa-id-card> Auth Service]
    end

    subgraph "Bus de Messages"
        KafkaBus((<fa:fa-random> Kafka))
        TopicProduct[Topic: product_events]
        TopicAuth[Topic: auth_events]
        KafkaBus --- TopicProduct
        KafkaBus --- TopicAuth
    end

    subgraph "Services Consommateurs"
        SearchSvc[<fa:fa-search> Search Service]
        OrderSvc[<fa:fa-receipt> Order Service]
    end

    ProductSvc -- "Événement 'PRODUCT_UPDATED'" --> TopicProduct
    AuthSvc -- "Événement 'USER_CREATED'" --> TopicAuth

    TopicProduct -- "Abonnement" --> SearchSvc
    TopicProduct -- "Abonnement" --> OrderSvc
    TopicAuth -- "Abonnement" --> OrderSvc

    SearchSvc -- "Met à jour l'index" --> Elasticsearch[(<fa:fa-search-plus> Elasticsearch)]
    OrderSvc -- "Met à jour les données dénormalisées" --> OrderDB[(<fa:fa-database> Order DB)]
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Mermaid
IGNORE_WHEN_COPYING_END
4.2. Découverte de Services via HashiCorp Consul

Rôle : Annuaire centralisé et dynamique des services.

Mécanisme : Au démarrage, chaque microservice contacte l'agent Consul et s'enregistre, fournissant son nom, son adresse IP (obtenue via la Downward API de Kubernetes), son port et un endpoint de health check. Lorsqu'un service (comme la Passerelle API) a besoin de communiquer avec un autre, il interroge Consul pour obtenir une liste des instances saines disponibles, éliminant tout besoin d'adresses IP codées en dur.

4.3. Indexation et Recherche via Elasticsearch

Rôle : Moteur de recherche textuelle.

Mécanisme : Le search-service maintient un index (products) dont le mapping est optimisé pour la recherche : les champs textuels (name, description) utilisent des analyseurs pour la tokenisation et la lemmatisation, tandis que les champs de filtrage (sku, category_slugs) sont de type keyword pour des correspondances exactes. Les variantes et leurs prix sont stockés dans un champ nested pour permettre des requêtes précises sur des sous-objets.

4.4. Gestion de Cache et de Données Éphémères via Redis

Rôle : Base de données en mémoire pour des accès à très faible latence.

Mécanisme : Le cart-service l'utilise pour stocker les paniers. Le choix de Redis est justifié par la nature volatile et non-critique (en termes de persistance à long terme) des paniers d'achat, où la performance prime sur la durabilité transactionnelle.

4.5. Persistance des Données via PostgreSQL

Rôle : Système de gestion de base de données relationnelle pour les données nécessitant une intégrité transactionnelle (utilisateurs, produits, commandes).

Mécanisme : Trois instances de base de données distinctes sont déployées, une pour chaque service concerné, concrétisant le pattern "Database per Service". Prisma ORM est utilisé pour la modélisation, les migrations de schéma et l'accès aux données.

5. Conteneurisation, Orchestration et Intégration Continue (CI/CD)
5.1. Conteneurisation avec Docker

Chaque service est encapsulé dans une image Docker légère et reproductible, construite à partir d'un Dockerfile standardisé utilisant une base node:18-alpine.

5.2. Orchestration avec Kubernetes (Kind)

L'application est déployée sur un cluster Kubernetes local (via Kind), utilisant des manifestes déclaratifs.

Objets Clés et leur Rôle :

Deployment : Gère le cycle de vie des Pods, assure le nombre de répliques souhaité et pilote les mises à jour sans interruption (rolling updates).

Service : Fournit une abstraction réseau stable (ex: auth-service-svc) pour accéder aux Pods, qui sont éphémères.

Init Container : Concept crucial pour la robustesse du déploiement. Ce conteneur spécialisé s'exécute avant le conteneur applicatif principal. Son unique rôle est d'exécuter npx prisma db push et npx prisma db seed, garantissant que la base de données est prête et à jour avant que le service ne tente de s'y connecter. Cela élimine les conditions de course au démarrage.

5.3. Pipeline CI/CD avec Jenkins

Le Jenkinsfile automatise entièrement le cycle de vie de l'application.

Étapes de la Pipeline :

Checkout : Récupération du code source.

Build Custom Docker Images : Construction des images Docker pour chaque service, taguées de manière unique avec l'ID du build (${env.BUILD_ID}).

Setup Kind Cluster : Création d'un cluster Kubernetes éphémère et pré-chargement de toutes les images Docker nécessaires pour accélérer le déploiement.

Deploy Application to Kind : Rendu du manifeste Kubernetes kubernetes-manifests.yaml avec les bons tags d'image via envsubst, puis application au cluster. La pipeline attend ensuite activement que tous les déploiements soient terminés avec succès (kubectl rollout status).

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

Ce projet démontre avec succès la mise en œuvre d'une architecture microservices complexe, fonctionnelle et alignée sur les standards industriels. La séparation des domaines, l'utilisation d'un bus d'événements pour la communication asynchrone et l'automatisation via CI/CD constituent une base solide pour une application d'e-commerce moderne et évolutive.

Pistes d'Amélioration et Perspectives :

Sécurité Avancée : Intégration d'un système de gestion de secrets (ex: HashiCorp Vault) pour remplacer les secrets en clair et mise en place de politiques de sécurité réseau plus strictes.

Observabilité : Déploiement d'une pile de monitoring (Prometheus, Grafana) et de logging centralisé (ELK Stack) pour une visibilité complète sur la performance et la santé du système.

Tests de Contrat : Mise en place de tests de contrat (ex: avec Pact) pour garantir que les modifications d'un service ne brisent pas les attentes de ses consommateurs, un enjeu majeur dans les architectures microservices.

Patterns de Résilience : Implémentation de patterns plus avancés comme le Circuit Breaker pour les appels synchrones (ex: commande -> stock) afin d'éviter les pannes en cascade.

Déploiement Cloud : Transition de Kind vers un service Kubernetes managé (EKS, GKE, AKS) pour un véritable déploiement en production.
