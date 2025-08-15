
### **Rapport Complet sur l'Implémentation des Fonctionnalités Post-Achat**

**Objet :** Définition de la logique métier, des flux de processus et des impacts architecturaux pour l'intégration des fonctionnalités d'annulation, de modification et de retour de commandes au sein de la plateforme e-commerce.

**Date :** 15 août 2025

**Auteur :** Gemini AI

---

#### **Introduction**

L'objectif de ce document est de fournir un cadre stratégique et technique pour le développement des fonctionnalités post-achat. Ces fonctionnalités sont critiques pour l'expérience client et l'efficacité opérationnelle. Nous analyserons chaque fonctionnalité en la liant directement à l'architecture microservices existante, aux schémas de base de données et aux flux de communication via Kafka.

---

### **1. Annulation de Commande**

L'annulation est la première fonctionnalité post-achat essentielle. Sa mise en œuvre doit être rigoureusement contrôlée pour garantir la cohérence des données de stock et de paiement.

#### **1.1 Logique Métier**

*   **Règles d'Annulation :**
    *   **Par le Client :** Un client peut annuler sa propre commande de manière autonome tant que celle-ci n'a pas atteint le statut `SHIPPED`. Les statuts éligibles sont donc `PENDING` et `PAID`. L'interface utilisateur, notamment le composant `OrderDetail.jsx`, devra masquer l'option d'annulation une fois ce seuil dépassé.
    *   **Par un Administrateur :** Un utilisateur disposant de la permission `update:order` peut annuler une commande à presque n'importe quelle étape avant la livraison effective. Les motifs incluent la suspicion de fraude, une rupture de stock imprévue, ou une demande client hors délai.
*   **Conséquences Systémiques :**
    *   **Gestion des Stocks :** L'annulation d'une commande doit déclencher la **restitution immédiate** des quantités réservées au stock vendable. C'est une opération critique.
    *   **Gestion des Paiements :** Si la commande avait le statut `PAID`, l'annulation doit **automatiquement initier un remboursement intégral** via le `Payment Service`.

#### **1.2 Flux de Processus (Diagramme de Séquence)**

Ce diagramme illustre le flux complet d'une annulation initiée par le client.

```mermaid
sequenceDiagram
    participant Client
    participant Frontend
    participant Gateway API
    participant Service Commandes
    participant Kafka
    participant Service Produits
    participant Service Paiements

    Client->>Frontend: Clique sur 'Annuler la Commande'
    Frontend->>Gateway API: PUT /orders/{id}/status (body: {status: "CANCELLED"})
    Gateway API->>Service Commandes: Forward la requête
    Service Commandes->>DB Commandes: Vérifie si statut est PENDING/PAID, puis UPDATE status = 'CANCELLED'
    Service Commandes->>Kafka: PUBLIE Événement 'ORDER_CANCELLED' (avec détails de la commande)
    Service Commandes-->>Gateway API: Réponse 200 OK (Succès)
    Gateway API-->>Frontend: Réponse 200 OK
    Frontend-->>Client: Affiche la confirmation d'annulation

    alt Si la commande était 'PAID'
        Kafka-->>Service Paiements: CONSOMME l'événement 'ORDER_CANCELLED'
        Note right of Service Paiements: Initie la logique de remboursement
        Service Paiements->>API Externe: Demande de remboursement
    end

    Kafka-->>Service Produits: CONSOMME l'événement 'ORDER_CANCELLED'
    Note right of Service Produits: Libère les articles réservés
    Service Produits->>DB Produits: UPDATE Variant SET stockQuantity = stockQuantity + N
```

#### **1.3 Cas Limites à Considérer**

*   **Condition de Course (Race Condition) :** Un client annule en même temps qu'un employé scanne le colis pour l'expédition. La transaction dans la base de données du `Order Service` doit être atomique pour garantir que seule la première opération réussit. L'autre doit recevoir une erreur claire (ex: "Impossible d'annuler, la commande vient d'être expédiée").
*   **Échec du Remboursement :** Si l'API de paiement retourne une erreur lors du remboursement, la commande doit être marquée avec un statut spécifique (ex: `CANCELLATION_ERROR`) pour alerter une équipe de support.

---

### **2. Modification de Commande**

La modification d'une commande en cours est une fonctionnalité d'une grande complexité.

**Recommandation Stratégique :** Ne pas implémenter la modification directe dans la version initiale. Adopter une politique claire et simple : **"Annuler la commande et en passer une nouvelle."**

#### **2.1 Justification de la Recommandation**

*   **Complexité des Paiements :**
    *   **Ajout d'article :** Nécessiterait une nouvelle autorisation de paiement du client pour le montant additionnel, ce qui est un défi technique et sécuritaire majeur.
    *   **Retrait d'article :** Nécessiterait un remboursement partiel, qui doit être supporté par le `Payment Service` et la passerelle de paiement.
*   **Complexité des Promotions :** Un changement dans le panier peut invalider des promotions appliquées, obligeant un recalcul complet et potentiellement confus pour le client.
*   **Alternative Simple :** Le flux "Annuler et Recommander" est universellement compris, ne présente pas de risques de paiement et s'appuie sur des fonctionnalités déjà existantes (annulation, création de commande).

#### **2.2 Flux de Processus Utilisateur Recommandé (Corrigé)**

Ce diagramme illustre l'approche recommandée, qui guide l'utilisateur vers une solution simple.

```mermaid
graph TD
    A[Client souhaite modifier sa commande] --> B{Commande déjà expédiée?};
    B -- Oui --> C["Modification impossible.<br>L'utilisateur est invité à procéder<br>à un retour après réception."];
    B -- Non --> D["L'utilisateur est guidé vers<br>l'annulation de sa commande actuelle"];
    D --> E["Le système traite l'annulation<br>(remboursement et remise en stock)"];
    E --> F["L'utilisateur est invité à créer<br>un nouveau panier et à passer<br>une nouvelle commande corrigée"];
```

---

### **3. Retours de Commande (Processus RMA)**

Ce processus est fondamental pour la confiance client et gère le cycle de vie d'un produit après sa livraison.

#### **3.1 Logique Métier**

1.  **Éligibilité au Retour :** Chaque produit doit avoir des règles de retour claires.
    *   **Attribut `isReturnable` (booléen) :** Définit si un article peut être retourné.
    *   **Attribut `returnPeriodDays` (entier) :** Définit la fenêtre de retour (ex: 30 jours après la date de livraison).
2.  **Processus RMA (Return Merchandise Authorization) :**
    *   **Étape 1 - Demande :** Le client initie une demande depuis son historique de commandes (`OrderDetail.jsx`).
    *   **Étape 2 - Approbation :** Un admin examine la demande et peut l'approuver ou la rejeter.
    *   **Étape 3 - Réception & Inspection :** L'entrepôt reçoit l'article retourné et vérifie son état.
    *   **Étape 4 - Résolution :** Un remboursement (total/partiel), un crédit en magasin ou un échange est effectué.

#### **3.2 Diagramme d'État d'une Demande de Retour**

Ce diagramme montre les transitions possibles pour une demande de retour.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> Demandé: Le client soumet sa demande
    Demandé --> Approuvé: L'admin accepte la demande
    Demandé --> Rejeté: L'admin refuse
    Approuvé --> En_Transit: Le client expédie le colis retour
    En_Transit --> Reçu_pour_Inspection: L'entrepôt scanne le colis reçu
    Reçu_pour_Inspection --> Remboursé: Article validé, remboursement déclenché
    Reçu_pour_Inspection --> Litige: Article non conforme ou endommagé
    Litige --> Remboursé
    Remboursé --> [*]
    Rejeté --> [*]
```

---

### **4. Modifications Proposées du Modèle de Données (ERD)**

Pour supporter ces nouvelles logiques, les schémas de vos bases de données doivent être étendus.

#### **4.1 Diagramme Entité-Relation - Modifications pour `product-service`**

```mermaid
erDiagram
    Product {
        string id PK
        string sku UK
        string name
        boolean isReturnable "NOUVEAU: Indique si le produit est retournable"
        int returnPeriodDays "NOUVEAU: Durée en jours de la politique de retour"
    }

    Variant {
        string id PK
        string productId FK
        int stockQuantity
    }

    StockMovement {
        string id PK
        string variantId FK
        int changeQuantity
        StockMovementType type "MISE À JOUR: Doit inclure 'RETURN'"
    }

    Product ||--o{ Variant : "possède"
    Variant ||--o{ StockMovement : "a"
```

#### **4.2 Diagramme Entité-Relation - Nouveaux Modèles pour `order-service`**

```mermaid
erDiagram
    Order {
        string id PK
        OrderStatus status
        decimal totalAmount
    }

    OrderItem {
        string id PK
        string orderId FK
        string productId
    }

    ReturnRequest {
        string id PK "NOUVEAU MODÈLE"
        string orderId FK
        ReturnStatus status
        string reason
    }

    ReturnRequestItem {
        string id PK "NOUVEAU MODÈLE"
        string returnRequestId FK
        string orderItemId FK
        int quantity
    }

    Order ||--o{ OrderItem : "contient"
    Order ||--o{ ReturnRequest : "peut générer"
    ReturnRequest ||--o{ ReturnRequestItem : "concerne"
    OrderItem }o--|| ReturnRequestItem : "est l'objet de"
```

---

### **5. Flux de Communication Inter-Services pour un Retour**

Le processus de retour est un excellent exemple de la collaboration asynchrone entre vos services.

```mermaid
sequenceDiagram
    participant Frontend
    participant Gateway API
    participant Service Commandes
    participant Kafka
    participant Service Produits
    participant Service Paiements

    Note over Frontend: Un admin valide un retour reçu et inspecté
    Frontend->>Gateway API: POST /orders/returns/{returnId}/complete
    Gateway API->>Service Commandes: Forward la requête
    Service Commandes->>DB Commandes: UPDATE ReturnRequest SET status = 'COMPLETED'
    Service Commandes->>Kafka: PUBLIE Événement 'ORDER_ITEM_RETURNED' (payload: {items, orderId})

    Kafka-->>Service Produits: CONSOMME 'ORDER_ITEM_RETURNED'
    Note right of Service Produits: Crée un mouvement de stock de type 'RETURN' pour augmenter le stock
    Service Produits->>DB Produits: INSERT INTO StockMovement ...

    Kafka-->>Service Paiements: CONSOMME 'ORDER_ITEM_RETURNED'
    Note right of Service Paiements: Calcule le montant à rembourser
    Service Paiements->>API Externe: Initie un remboursement partiel
```

---

### **Conclusion et Recommandations**

1.  **Annulation :** **À implémenter.** C'est une fonctionnalité attendue et gérable. La priorité est de garantir l'atomicité des opérations de stock et de remboursement via des transactions et des événements fiables.

2.  **Modification :** **À reporter.** Adopter la politique **"Annuler et Recommander"** pour réduire drastiquement la complexité et les risques, tout en offrant une solution fonctionnelle aux utilisateurs.

3.  **Retours :** **À implémenter.** C'est un processus complexe mais non négociable pour un e-commerce moderne. Il nécessite les évolutions de schéma proposées et une orchestration événementielle rigoureuse pour maintenir la cohérence des données à travers les services.

L'adoption de ces stratégies permettra d'enrichir la plateforme de manière robuste et maîtrisée, en capitalisant sur les forces de votre architecture microservices.
