

## 1. Philosophie et principes fondamentaux
Lureva repose sur un modèle de **divulgation nulle de connaissance (Zero-knowledge)**. L'infrastructure serveur (PostgreSQL, Object Storage) est conçue comme un environnement "hostile" : elle ne stocke que des données chiffrées et ne possède aucun moyen mathématique de les déchiffrer.

Notre cryptographie est **hybride et résistante aux ordinateurs quantiques** :
*   **Symétrique :** AES-256-GCM pour le chiffrement des données (fichiers, emails, CRDTs).
*   **Asymétrique :** combinaison de **X25519** (Standard) et **ML-KEM 768** (post-quantique / standard FIPS 204 du NIST).

## 2. Hiérarchie des clés (key hierarchy)
La sécurité de l'utilisateur repose sur un trousseau de clés géré localement dans son navigateur (en mémoire volatile).

1.  **Mot de passe utilisateur (ou sortie PRF des passkeys)** : c'est le secret racine. Il n'est jamais envoyé au serveur.
2.  **Master account key (MAK)** : une clé symétrique AES-256 bits unique générée à la création du compte. Elle est stockée chiffrée sur le serveur (enveloppée par le secret racine).
3.  **Hybrid keypair (HKP)** : la paire de clés asymétriques hybride (Publique/Privée) de l'utilisateur. La clé publique est distribuée en clair, la clé privée est chiffrée par la **MAK**.

## 3. Mécanismes d'authentification

### 3.1. Connexion par mot de passe (protocole OPAQUE)
Lureva n'envoie jamais le mot de passe au serveur et n'utilise pas de simples "hash". Nous utilisons le protocole **OPAQUE (aPKI)**.
1. Le client masque son mot de passe et l'envoie au serveur.
2. Le serveur effectue une opération mathématique aveugle à l'aide de sa propre clé secrète et renvoie le résultat.
3. Le client dérive une clé cryptographique forte pour s'authentifier.
4. Une fois authentifié, le client utilise localement son mot de passe pour déchiffrer la **MAK**, qui déchiffrera la **HKP**.

### 3.2. Connexion passwordless (WebAuthn passkeys + extension PRF)
Pour le 1-click login, Lureva utilise les passkeys avec l'extension **PRF (Pseudo-Random Function)**.
1. Lors du scan biométrique (FaceID, TouchID, YubiKey), l'authentificateur génère une signature *et* un secret symétrique déterministe (PRF Output).
2. Le serveur valide la signature WebAuthn.
3. Le navigateur utilise la sortie PRF brute (qui ne quitte jamais l'appareil) pour générer une clé HKDF qui va directement déchiffrer la **MAK**.

## 4. Partage et collaboration zero-knowledge

### 4.1. Partage utilisateur à utilisateur (P2P asynchrone)
Quand Alice veut partager une *Liste* ou un *Fichier* avec Bob :
1. Alice possède la clé symétrique AES de la Liste (ListKey).
2. L'application d'Alice récupère la **Clé Publique Hybride (HKP)** de Bob depuis le serveur.
3. Alice chiffre la *ListKey* avec la clé publique de Bob et l'envoie au serveur.
4. Bob se connecte, récupère l'enveloppe, et la déchiffre avec sa propre clé privée. Le serveur n'a jamais vu la *ListKey*.

### 4.2. Cas d'entreprise (enterprise key escrow)
Dans une organisation B2B (Plan business), l'administrateur a le droit légal d'accéder aux données des employés (en cas de départ, perte de mot de passe, ou audit).
1. L'organisation possède une **Org Keypair** (générée par le créateur de l'org).
2. Lors de la création d'un compte employé (Managed User), la **HKP Privée** de l'employé est chiffrée deux fois :
   * Une fois pour l'employé (avec son mot de passe).
   * Une fois pour l'Organisation (chiffrée avec la clé publique de l'Organisation).
3. Ce "Key Escrow" garantit que l'employé a un environnement privé, tout en permettant à l'Admin (qui possède la clé privée de l'Org) de recouvrer les accès sans rompre le modèle Zero-Knowledge vis-à-vis de Lureva.

## 5. Protection des sessions (DPoP)
Pour empêcher le vol de cookies de session (Token Hijacking), Lureva implémente **DPoP (Demonstrating Proof-of-Possession)**.
Lors du login, l'appareil génère une paire de clés éphémères non-extractibles. Chaque requête HTTP envoyée au serveur doit être signée par cette clé. Un attaquant qui volerait le JWT (cookie) sans pouvoir extraire la clé matérielle de l'appareil verra toutes ses requêtes rejetées.

---
