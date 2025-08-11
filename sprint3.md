Absolument. Je comprends parfaitement. Ma version précédente était inadéquate, manquait de profondeur et n'a pas su intégrer correctement le contexte que vous avez fourni. Je vous présente mes excuses pour cette paresse.

Je vais rédiger un rapport entièrement nouveau, beaucoup plus détaillé, technique et exhaustif, en respectant scrupuleusement toutes vos nouvelles directives. Le document mettra en avant la complexité et la justification des choix techniques, en particulier l'intégration d'Istio comme un pilier pour des travaux de recherche futurs.

Voici la version complète et approfondie.

Rapport Technique Approfondi : Industrialisation d'une Plateforme Microservices sur Kubernetes avec un Service Mesh pour le Déploiement et la Recherche

Phase du Projet : Industrialisation, Déploiement et Préparation pour la Recherche Avancée

Présenté par :
HARCHE Samir

Sous la direction de :
Dr. Abdelhak Djamel Seriai

Abstract

Ce rapport technique documente la seconde phase majeure du projet de plateforme e-commerce, marquant sa transition d'un Produit Minimum Viable (MVP), initialement orchestré avec Docker Compose, vers une architecture industrialisée et prête pour la production. L'objectif principal est la conception et la mise en œuvre d'une infrastructure de déploiement robuste, scalable et entièrement automatisée, en utilisant des technologies de pointe. Le cœur de cette infrastructure repose sur Kubernetes (K8s), avec des clusters locaux à haute-fidélité provisionnés par Kind. Une attention particulière est portée à l'intégration du Service Mesh Istio, qui transcende le simple routage pour devenir une fondation critique pour des travaux de recherche futurs dans le cadre d'un second projet de Master intitulé "Techniques IA dans les applications microservices". Ce rapport dissèque en profondeur les manifestes YAML de Kubernetes, en détaillant des patterns essentiels tels que les Init Containers pour la fiabilisation des dépendances et la séparation stratégique des ressources. Il analyse également le pipeline CI/CD orchestré par Jenkins, qui automatise l'ensemble du cycle de vie du déploiement. Le document conclut en présentant une infrastructure non seulement apte au déploiement, mais également conçue comme une plateforme expérimentale pour des algorithmes avancés d'équilibrage de charge pilotés par l'IA.

Table des Matières

Introduction : Au-delà de l'MVP, vers une Plateforme de Production et de Recherche

Contexte : La Continuation d'un Travail Précédent

La Stratégie à Double Environnement : Le "Inner" et le "Outer Loop"

Objectif Stratégique d'Istio : Un Pilier pour la Recherche en IA

Architecture de la Plateforme sur Kubernetes

Les Piliers Technologiques : Kubernetes, Kind, Istio

Vue Macroscopique du Cluster et des Flux de Trafic

Anatomie d'un Pod dans le Service Mesh

Analyse Détaillée des Manifestes de Déploiement

Stratégie de Fichiers : infra-manifests.yaml vs app-manifests.yaml

Le Triptyque Fondamental : Deployment, ReplicaSet et Pod

Fiabilisation des Dépendances avec le Pattern Init Container

Le Service Mesh Istio : Un Contrôle Avancé du Trafic

Le Point d'Entrée : Gateway et VirtualService

Cas d'Usage : Manipulation de l'URI avec rewrite

Le Pipeline d'Intégration et de Déploiement Continus (CI/CD)

Provisioning de l'Environnement avec setup-kind.sh

Orchestration du Déploiement avec le Jenkinsfile

Analyse Détaillée des Étapes (Stages)

Le Mécanisme de Mise à Jour : kubectl rollout restart

Conclusion et Synergie avec la Recherche en IA

Bilan des Réalisations Techniques

Travaux Futurs : Vers un Équilibrage de Charge Intelligent

1. Introduction : Au-delà de l'MVP, vers une Plateforme de Production et de Recherche
1.1. Contexte : La Continuation d'un Travail Précédent

Ce document s'inscrit dans la continuité d'une première phase de développement qui avait permis de valider la conception d'une plateforme e-commerce via un Produit Minimum Viable (MVP). Cette phase initiale reposait sur Docker Compose pour orchestrer les différents microservices, une approche efficace pour prototyper et valider les interactions fondamentales. Le présent rapport détaille la phase d'industrialisation, qui vise à transformer ce MVP en une application déployable sur une infrastructure robuste, scalable et résiliente, simulant les conditions d'une mise en production.

1.2. La Stratégie à Double Environnement : Le "Inner" et le "Outer Loop"

Il est crucial de noter que Docker Compose n'est pas abandonné. Une stratégie à deux volets a été adoptée pour optimiser le cycle de vie du développement :

Inner Loop (Développement Quotidien) : Docker Compose reste l'outil de prédilection pour le développement local. Sa simplicité et la rapidité du hot-reloading offrent une vélocité maximale au développeur pour écrire, tester et déboguer son code.

Outer Loop (Intégration et Déploiement) : Pour l'intégration continue, les tests de non-régression et le déploiement final, la plateforme Kubernetes est utilisée. Elle seule peut offrir les garanties de haute disponibilité, d'orchestration et de gestion de trafic requises.

code
Mermaid
download
content_copy
expand_less

graph TD
    subgraph "Inner Loop (Développeur)"
        A[Coder dans l'IDE] --> B{Lancer via `dev.sh` / `docker-compose`};
        B --> C[Tester localement];
        C --> D{Modifier le code};
        D -.-> A;
    end

    subgraph "Outer Loop (CI/CD - Jenkins)"
        E[Push sur Git] --> F{Déclenchement du Pipeline Jenkins};
        F --> G[Build des images Docker];
        G --> H[Déploiement sur le Cluster Kubernetes (Kind)];
        H --> I[Tests d'Intégration / E2E];
        I --> J[Déploiement en Production];
    end

    style A fill:#cde4ff
    style F fill:#d5e8d4

Figure 1.1 : Distinction entre le cycle de développement rapide ("Inner Loop") et le cycle de déploiement industrialisé ("Outer Loop").

1.3. Objectif Stratégique d'Istio : Un Pilier pour la Recherche en IA

Le choix d'intégrer le Service Mesh Istio n'est pas anodin et dépasse les simples besoins de routage. Cette décision est fondamentale et directement liée à un second projet de Master : "Techniques IA dans les applications microservices". Istio, par sa capacité à intercepter et contrôler finement tout le trafic L7 entre les microservices, fournit les APIs de télémétrie et de contrôle nécessaires pour expérimenter et implémenter des algorithmes d'équilibrage de charge intelligents. L'infrastructure décrite dans ce rapport n'est donc pas seulement une finalité en soi, mais aussi une plateforme expérimentale conçue pour permettre cette recherche avancée.

2. Architecture de la Plateforme sur Kubernetes
2.1. Les Piliers Technologiques : Kubernetes, Kind, Istio

Kubernetes (K8s): L'orchestrateur. Il gère l'état désiré de nos applications, leur mise à l'échelle (scaling), leur auto-réparation (self-healing) et la découverte de services (service discovery) via son DNS interne.

Kind (Kubernetes in Docker): L'outil de provisioning de cluster local. Il permet de créer un cluster Kubernetes multi-nœuds en quelques minutes, offrant un environnement à très haute-fidélité pour tester nos manifestes de déploiement avant de les appliquer sur un cluster de production (ex: GKE, AKS, EKS).

Istio: Le Service Mesh. Il est déployé sur Kubernetes et injecte un proxy sidecar (basé sur Envoy) dans chaque Pod applicatif. Ce proxy intercepte tout le trafic entrant et sortant, permettant de mettre en œuvre des politiques de routage, de sécurité et de collecter des métriques détaillées sans modifier une seule ligne de code applicatif.

2.2. Vue Macroscopique du Cluster et des Flux de Trafic

Le diagramme ci-dessous illustre l'organisation globale du cluster, des namespaces et le cheminement d'une requête externe à travers les différentes couches jusqu'à un microservice.

code
Mermaid
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
graph TD
    subgraph "Monde Extérieur"
        User[Client / Utilisateur]
    end

    User -- HTTPS Request --> HostPort[Machine Hôte: localhost:13000]

    subgraph "Cluster `Kind`"
        HostPort -- "Port Mapping" --> IngressGatewayService[Service `NodePort` <br> `istio-ingressgateway`]

        subgraph "Namespace: `istio-system`"
            IngressGatewayService --> IngressGatewayPod[Pod <br> `istio-ingressgateway`]
            IstioD[Pod <br> `istiod` (Control Plane)]
        end

        subgraph "Namespace: `lirmm-services` (istio-injection=enabled)"
            IngressGatewayPod -- "Transmet le trafic via mTLS" --> LirmmGateway[Istio `Gateway` Resource]
            LirmmGateway -- "Applique les règles du" --> VirtualService[Istio `VirtualService` Resource]

            subgraph "Couche Applicative"
                VirtualService -- "Route /auth/**" --> AuthService[Service `auth-service-svc`]
                AuthService --> AuthPod[Pod `auth-service`]
                VirtualService -- "Route /products/**" --> ProductService[Service `product-service-svc`]
                ProductService --> ProductPod[Pod `product-service`]
                VirtualService -- "..." --> EtcPods[...]
            end

            subgraph "Couche Persistance & Messaging (sans sidecar)"
                AuthPod -- "TCP" --> AuthDB[Service `auth-db-svc`]
                ProductPod -- "TCP" --> ProductDB[Service `product-db-svc`]
                AuthPod -- "Publie event" --> Kafka[Service `kafka-svc`]
            end
        end
    end

    IstioD -- "Configure les sidecars" --> AuthPod
    IstioD -- "Configure les sidecars" --> ProductPod

    style User fill:#f9f

Figure 2.1 : Architecture détaillée du cluster Kind avec Istio.

2.3. Anatomie d'un Pod dans le Service Mesh

L'activation de l'injection Istio sur le namespace lirmm-services modifie fondamentalement la structure de nos Pods. Chaque Pod applicatif contient désormais deux conteneurs.

code
Mermaid
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
graph TD
    subgraph "Pod: `payment-service`"
        direction LR
        subgraph "Conteneurs"
            AppContainer[Container: `payment-service` <br> (Node.js)];
            ProxyContainer[Container: `istio-proxy` (Envoy) <br> injecté automatiquement];
        end

        subgraph "Réseau du Pod (NetNS partagé)"
            TrafficIn(Trafic Entrant) --> ProxyContainer;
            ProxyContainer -- "Applique les politiques (mTLS, routage)" --> AppContainer;
            AppContainer -- "Trafic Sortant" --> ProxyContainer;
            ProxyContainer -- "Applique les politiques" --> TrafficOut(Trafic Sortant);
        end
    end
    style ProxyContainer fill:#ccf,stroke:#333

Figure 2.2 : Structure interne d'un Pod avec le proxy sidecar Istio qui intercepte tout le trafic.

3. Analyse Détaillée des Manifestes de Déploiement
3.1. Stratégie de Fichiers : infra-manifests.yaml vs app-manifests.yaml

La séparation des manifestes est une pratique essentielle pour la gestion de la configuration :

infra-manifests.yaml: Définit les composants tiers Stateful (bases de données PostgreSQL, Kafka, Redis, Elasticsearch). Leur cycle de vie est long. Ils sont explicitement exclus du Service Mesh avec l'annotation sidecar.istio.io/inject: "false" car leurs protocoles ne sont pas toujours basés sur HTTP et pour éviter une surcharge de performance inutile.

app-manifests.yaml: Définit nos microservices Stateless. Leur cycle de vie est court, lié aux itérations de développement. C'est ce fichier qui est manipulé par le pipeline CI/CD.

3.2. Le Triptyque Fondamental : Deployment, ReplicaSet et Pod

Pour chaque microservice, nous utilisons une ressource Deployment. C'est une abstraction de haut niveau qui gère le cycle de vie des Pods.

code
Mermaid
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
graph TD
    subgraph "Niveau Déclaratif (Ce que l'on applique)"
        D[Deployment: `payment-service-deployment` <br> spec.replicas = 2 <br> spec.template = {Pod Spec}]
    end
    
    subgraph "Niveau Contrôleur (Ce que Kubernetes gère)"
        RS[ReplicaSet <br> Gère 2 répliques]
    end

    subgraph "Niveau Exécution (Ce qui tourne sur les Nœuds)"
        P1[Pod: `payment-service-xxxx`]
        P2[Pod: `payment-service-yyyy`]
    end

    D -- "Crée / Met à jour" --> RS;
    RS -- "Garantit l'état désiré en créant" --> P1;
    RS -- "Garantit l'état désiré en créant" --> P2;

Figure 3.1 : Relation entre les objets Deployment, ReplicaSet et Pod dans Kubernetes.

3.3. Fiabilisation des Dépendances avec le Pattern Init Container

Un Pod ne doit pas être considéré comme "prêt" (Ready) si ses dépendances ne le sont pas. Le pattern Init Container est la solution native de Kubernetes à ce problème. Pour chaque service nécessitant une migration de base de données (ex: auth-service, order-service), nous définissons un initContainer qui exécute la commande npx prisma db push.

code
Yaml
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
# Extrait de app-manifests.yaml pour auth-service-deployment
spec:
  template:
    spec:
      initContainers:
      - name: auth-db-migrate
        image: lirmm-ecommerce/auth-service:latest
        command: ["sh", "-c", "npx prisma db push --schema=./prisma/schema.prisma --accept-data-loss && npx prisma db seed --schema=./prisma/schema.prisma"]
        env:
        - name: DATABASE_URL
          value: "postgresql://postgres:postgres@auth-db-svc:5432/auth_db?schema=public"
      containers:
      - name: auth-service
        # ...

Le conteneur principal (auth-service) ne démarrera que si et seulement si le initContainer (auth-db-migrate) se termine avec un code de sortie 0.

4. Le Service Mesh Istio : Un Contrôle Avancé du Trafic
4.1. Le Point d'Entrée : Gateway et VirtualService

Le trafic externe est géré par ce couple de ressources Istio :

Gateway (lirmm-gateway) : Déclare un point d'entrée sur le pod istio-ingressgateway, spécifiant le port (80) et les domaines (*). C'est la porte.

VirtualService (main-routing-vs) : S'attache à la Gateway et agit comme un routeur intelligent, dirigeant le trafic vers les services Kubernetes internes en fonction de règles de match (ex: préfixe de l'URI).

4.2. Cas d'Usage : Manipulation de l'URI avec rewrite

Istio permet de découpler l'URL publique de l'API interne du microservice. Par exemple, une requête publique sur /products/ peut être réécrite en / avant d'être envoyée au product-service.

code
Yaml
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
# Extrait du VirtualService dans app-manifests.yaml
spec:
  http:
  - match:
    - uri:
        prefix: /products/
    rewrite:
      uri: /  # Réécrit /products/some/path en /some/path
    route:
    - destination:
        host: product-service-svc.lirmm-services.svc.cluster.local
        port:
          number: 3003
5. Le Pipeline d'Intégration et de Déploiement Continus (CI/CD)
5.1. Provisioning de l'Environnement avec setup-kind.sh

Ce script automatise entièrement la création d'un environnement de test vierge. Ses actions clés sont :

kind delete cluster / kind create cluster: Assure un cluster propre à chaque exécution.

istioctl install --set profile=demo: Installe Istio avec ses composants.

kubectl patch svc istio-ingressgateway: Configure le Service de la Gateway en NodePort pour l'exposer sur localhost.

kubectl create namespace / kubectl label namespace ... istio-injection=enabled: Crée le namespace pour nos applications et active l'injection automatique des sidecars.

5.2. Orchestration du Déploiement avec le Jenkinsfile
5.2.1. Analyse Détaillée des Étapes (Stages)

Le Jenkinsfile définit un pipeline déclaratif qui automatise le déploiement des applications sur le cluster Kubernetes.

Stage('Build & Load Application Images'): Cette étape itère sur une liste de microservices. Pour chacun, elle exécute docker build pour créer l'image, puis kind load docker-image pour la charger directement dans la mémoire des nœuds du cluster Kind, évitant un push/pull coûteux vers une registry.

Stage('Deploy Application'): Cette étape orchestre le déploiement sur le cluster.

kubectl apply -f ${env.APP_MANIFEST_FILE}: Kubernetes compare l'état désiré dans le fichier avec l'état actuel et n'applique que les changements.

kubectl rollout restart deployment ...: La commande cruciale qui force le redémarrage des Pods pour qu'ils utilisent les nouvelles images.

kubectl wait --for=condition=Available ...: Le pipeline attend que tous les Pods soient opérationnels avant de se déclarer en succès.

5.2.2. Le Mécanisme de Mise à Jour : kubectl rollout restart

Lorsque l'on utilise des tags d'image statiques comme :latest, modifier l'image ne suffit pas à déclencher une mise à jour. Le rollout restart est la solution.

code
Mermaid
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
sequenceDiagram
    participant Jenkins as Pipeline Jenkins
    participant K8s as Kubernetes API Server
    participant RS_Old as ReplicaSet (v1)
    participant RS_New as ReplicaSet (v2)

    Jenkins->>K8s: 1. `kubectl rollout restart deployment/payment-service`
    K8s->>RS_Old: 2. Met à l'échelle à 0 (progressivement)
    K8s->>+RS_New: 3. Crée un nouveau ReplicaSet (v2) avec le même template de Pod
    RS_New->>K8s: 4. Crée un nouveau Pod (P_new1)
    Note right of RS_New: Le nouveau Pod tire la nouvelle image Docker
    RS_Old->>K8s: 5. Supprime un ancien Pod (P_old1)
    loop Jusqu'à ce que toutes les répliques soient remplacées
        RS_New->>K8s: Crée Pod (P_new2)
        RS_Old->>K8s: Supprime Pod (P_old2)
    end
    K8s-->>-RS_New: Déploiement terminé

Figure 5.1 : Diagramme de séquence d'un rolling update initié par rollout restart.

6. Conclusion et Synergie avec la Recherche en IA
6.1. Bilan des Réalisations Techniques

Cette phase d'industrialisation a permis de construire une infrastructure de déploiement CI/CD complète, robuste et alignée sur les standards de l'industrie. La stratégie à deux volets (Docker Compose / Kubernetes) maximise la productivité tout en garantissant la fiabilité des déploiements. Plus important encore, l'architecture a été conçue pour être extensible.

6.2. Travaux Futurs : Vers un Équilibrage de Charge Intelligent

L'infrastructure actuelle est la fondation indispensable pour la thèse sur les techniques d'IA. Les prochaines étapes consisteront à :

Exploiter la Télémétrie d'Istio : Collecter les métriques avancées (latence p99, taux d'erreur, etc.) via Prometheus, qui est déjà installé.

Développer un Contrôleur Externe : Créer un service qui interroge ces métriques et héberge les algorithmes d'IA (ex: apprentissage par renforcement).

Piloter Istio par API : Ce contrôleur modifiera dynamiquement les ressources Istio (ex: poids dans un VirtualService) pour ajuster en temps réel la répartition de charge en fonction des prédictions de l'algorithme, bien au-delà du simple Round Robin.

code
Mermaid
download
content_copy
expand_less
IGNORE_WHEN_COPYING_START
IGNORE_WHEN_COPYING_END
graph TD
    subgraph "Boucle de Contrôle IA"
        Telemetry[Prometheus/Istio Telemetry] -- "1. Métriques (Latence, CPU)" --> AI_Controller[Contrôleur IA];
        AI_Controller -- "2. Décision (ex: poids)" --> Istio_API[API de contrôle Istio];
        Istio_API -- "3. Applique nouvelle politique" --> VS[VirtualService];
    end
    subgraph "Plan de Données (Data Plane)"
        VS -- "Répartit le trafic (ex: 80%/20%)" --> V1[Pods v1: `product-service`];
        VS -- "Répartit le trafic (ex: 80%/20%)" --> V2[Pods v2: `product-service`];
    end

    V1 -- "Feedback" --> Telemetry;
    V2 -- "Feedback" --> Telemetry;

Figure 6.1 : Architecture cible pour l'équilibrage de charge piloté par l'IA.
