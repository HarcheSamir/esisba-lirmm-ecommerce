### **Rapport d'Architecture Système : RBAC et Gestion du Cycle de Vie des Utilisateurs**

#### **Résumé Exécutif**

Le système met en œuvre une architecture robuste et événementielle pour la gestion de l'identité des utilisateurs et du contrôle d'accès. Il est fondé sur un modèle granulaire de Contrôle d'Accès Basé sur les Rôles (RBAC), où les permissions sont l'unité fondamentale d'autorité. Ces permissions sont agrégées en rôles, qui sont ensuite assignés aux utilisateurs. Le cycle de vie de la création d'un utilisateur est géré par un flux d'invitation asynchrone et sécurisé, garantissant que le service principal `auth-service` est découplé des préoccupations externes telles que les notifications par e-mail. Cette conception est sécurisée, évolutive et conforme aux meilleures pratiques modernes des microservices.

---

### **Partie 1 : Le Modèle de Données Principal**

L'ensemble du système RBAC repose sur un modèle de données clair et robuste au sein de la base de données PostgreSQL du service `auth-service`. Les relations entre `User`, `Role` et `Permission` constituent le fondement de toute la logique de sécurité.

#### **Diagramme Entité-Relation (DER)**

Ce diagramme illustre le schéma de la base de données. L'élément clé est la table de jonction `RolePermission`, qui crée une relation plusieurs-à-plusieurs entre les Rôles et les Permissions, permettant une flexibilité maximale.

```mermaid
erDiagram
    User {
        string id PK
        string name
        string email
        string password
        boolean isActive
        string invitationToken
        datetime invitationExpires
        string roleId FK
    }
    Role {
        string id PK
        string name
        string description
    }
    Permission {
        string id PK
        string name
        string description
    }
    RolePermission {
        string roleId FK
        string permissionId FK
    }

    User }|--|| Role : "est assigné à"
    Role ||--|{ RolePermission : "possède"
    Permission ||--|{ RolePermission : "appartient à"
```

**Détail du Modèle :**

1.  **Permission :** L'unité d'autorité la plus granulaire (ex : `create:product`, `read:user`). Celles-ci sont statiques et définies dans le script d'initialisation (seed script). Elles représentent une action unique qu'un utilisateur peut effectuer.
2.  **Rôle :** Une collection nommée de permissions (ex : "Superviseur", "Auditeur"). Un rôle définit une fonction professionnelle au sein du système.
3.  **Utilisateur :** Une entité individuelle à laquelle est assigné exactement un Rôle. L'utilisateur hérite de toutes les permissions accordées par ce rôle.
4.  **RolePermission :** La table de jonction qui lie les Rôles et les Permissions. C'est ce qui permet à un seul Rôle d'avoir plusieurs Permissions, et à une seule Permission de faire partie de plusieurs Rôles.

---

### **Partie 2 : Le Flux d'Invitation et de Création d'Utilisateur**

Il s'agit du processus le plus complexe, impliquant plusieurs services communiquant de manière asynchrone. Il est conçu pour être résilient et découplé. Un utilisateur est créé dans un état "en attente" et doit activer son propre compte.

#### **Diagramme de Séquence : Cycle de Vie Complet de l'Invitation**

Ce diagramme montre le processus de bout en bout, du clic initial de l'administrateur jusqu'à ce que le nouvel utilisateur soit pleinement actif dans le système.

```mermaid
sequenceDiagram
    acteur Admin
    participant API Gateway
    participant Auth Service
    participant Kafka Broker
    participant Notification Service
    acteur Nouvel Utilisateur

    Admin->>API Gateway: POST /users/invite (name, email, roleId)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Note over Auth Service: 1. Vérifier les permissions de l'admin (write:user).
    Auth Service->>Auth DB: INSERT INTO User (isActive: false, password: null)
    Note over Auth Service: 2. Générer et stocker un invitationToken unique.
    Auth Service->>Kafka Broker: PRODUCE event to 'auth_events' topic (type: USER_INVITED)
    Auth Service-->>API Gateway: 201 Created
    API Gateway-->>Admin: 201 Created
    deactivate Auth Service

    Kafka Broker-->>Notification Service: DELIVER USER_INVITED event

    activate Notification Service
    Note over Notification Service: 3. Analyser le payload de l'événement (email, token).
    Notification Service->>SMTP Server (Gmail): SEND invitation email
    deactivate Notification Service

    Nouvel Utilisateur->>Client Email: Ouvre l'email, clique sur le lien d'invitation
    Nouvel Utilisateur->>API Gateway: POST /auth/complete-invitation (token, password)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Note over Auth Service: 4. Trouver l'utilisateur par son invitationToken unique.
    Auth Service->>Auth DB: SELECT * FROM User WHERE invitationToken = ?
    Note over Auth Service: 5. Hacher le nouveau mot de passe.
    Auth Service->>Auth DB: UPDATE User SET password=?, isActive=true, invitationToken=null
    Note over Auth Service: 6. Générer un nouveau JWT pour l'utilisateur.
    Auth Service-->>API Gateway: 200 OK (with JWT)
    API Gateway-->>Nouvel Utilisateur: 200 OK (with JWT)
    deactivate Auth Service
```

---

### **Partie 3 : Le Flux d'Authentification et d'Autorisation**

Une fois qu'un utilisateur est actif, son identité et ses permissions sont gérées via des JSON Web Tokens (JWT).

#### **Flux 3.1 : Connexion de l'Utilisateur et Création du JWT**

Lorsqu'un utilisateur se connecte, le `auth-service` interroge la base de données pour obtenir toutes les permissions associées et les intègre directement dans le payload du JWT. Cela fait du JWT un "passeport" autonome contenant l'identité et les capacités de l'utilisateur.

```mermaid
sequenceDiagram
    acteur Utilisateur
    participant API Gateway
    participant Auth Service
    participant Auth DB

    Utilisateur->>API Gateway: POST /login (email, password)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Auth Service->>Auth DB: SELECT * FROM User WHERE email = ?
    Note over Auth Service: 1. Vérifier le hash du mot de passe avec bcrypt.
    Auth Service->>Auth DB: SELECT p.name FROM Permission p JOIN RolePermission rp ON p.id = rp.permissionId WHERE rp.roleId = ?
    Note over Auth Service: 2. Collecter toutes les chaînes de permission pour le rôle de l'utilisateur.
    Note over Auth Service: 3. Créer le payload du JWT : { id, name, email, role, permissions: ['read:product', ...] }
    Auth Service-->>API Gateway: 200 OK (with JWT)
    API Gateway-->>Utilisateur: 200 OK (with JWT)
    deactivate Auth Service
```

#### **Flux 3.2 : Requête API Authentifiée et Autorisée**

Lorsque le frontend effectue une requête vers une page protégée, il inclut le JWT. La logique frontend (HOC `WithPermission` et rendu conditionnel) utilise la liste des permissions contenue dans le JWT pour accorder ou refuser l'accès.

```mermaid
sequenceDiagram
    acteur Utilisateur
    participant App Frontend
    participant API Gateway
    participant Service Protégé

    Utilisateur->>App Frontend: Clique sur le lien "Produits"
    
    activate App Frontend
    Note over App Frontend: 1. Vérification barre latérale : hasPermission('read:product') ? -> OUI
    App Frontend->>API Gateway: GET /products (Authorization: Bearer <JWT>)
    deactivate App Frontend

    API Gateway->>Service Protégé: FORWARD request

    activate Service Protégé
    Note over Service Protégé: 2. Le middleware vérifie la signature du JWT (optionnel mais bonne pratique).
    Note over Service Protégé: 3. Bien que le frontend ait déjà vérifié, la route peut revérifier si nécessaire.
    Service Protégé->>Service Protégé: Exécute la logique métier (ex: récupérer les produits)
    Service Protégé-->>API Gateway: 200 OK (with product data)
    API Gateway-->>App Frontend: 200 OK (with product data)
```

---

### **Partie 4 : Implémentation Frontend**

Le frontend met en œuvre un modèle de sécurité à deux niveaux pour offrir une expérience utilisateur fluide.

1.  **Sécurité au Niveau de la Route :** Le HOC `WithPermission` agit comme un gardien strict pour des pages entières. Si un utilisateur essaie d'accéder directement à une URL, ce HOC vérifie ses permissions avant d'afficher le composant de la page.
2.  **Sécurité au Niveau du Composant :** Au sein d'une page, les éléments de l'interface utilisateur (boutons, champs de formulaire, liens) sont affichés de manière conditionnelle à l'aide de la fonction d'aide `hasPermission` du `authStore`. Cela garantit qu'un utilisateur ne voit jamais une action qu'il n'est pas autorisé à effectuer.

#### **Organigramme de la Logique Frontend**

Ce diagramme montre le processus de décision sur le frontend pour l'affichage de tout contenu protégé.

```mermaid
graph TD
    A[L'utilisateur tente d'accéder à une page ou de voir un composant] --> B{Est Authentifié ?<br/>Le token existe et est valide};
    B -- Non --> C[Redirection vers la page de Connexion];
    B -- Oui --> D{Permission Requise ?};
    D -- Non --> F[Afficher le Contenu];
    D -- Oui --> E{L'utilisateur a la permission requise ?};
    E -- Non --> G[Masquer le Composant OU<br/>Redirection vers Tableau de bord/Erreur];
    E -- Oui --> F;
```

Ce système complet garantit que l'accès des utilisateurs est contrôlé de manière sécurisée, efficace et cohérente, depuis la base de données jusqu'au dernier pixel affiché dans le navigateur de l'utilisateur.
