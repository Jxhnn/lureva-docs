
# Rapport d'Architecture et de Sécurité - Lureva SaaS

## 1. Philosophie de Sécurité & Infrastructure
Lureva est conçu sur un modèle de sécurité **Zero-Knowledge (Divulgation Nulle de Connaissance)**. Le serveur agit comme un simple relai et espace de stockage de données chiffrées. À aucun moment l'infrastructure ne possède les clés permettant de lire les données des utilisateurs. 

L'architecture repose sur trois piliers fondamentaux :
1. **Souveraineté des données :** L'infrastructure (Base de données PostgreSQL et moteur d'exécution Bun) est hébergée sur un cloud souverain français (ex: Scaleway / OVHcloud). Cela garantit que les métadonnées (qui communique avec qui, à quelle heure) sont protégées par le RGPD et totalement à l'abri des lois extraterritoriales américaines (CLOUD Act, FISA).
2. **Cryptographie Hybride de pointe :** 
   * Asymétrique : Chiffrement Post-Quantique (ML-KEM 768) combiné à l'algorithme classique X25519.
   * Symétrique : AES-256-GCM.
3. **Authentification de nouvelle génération :** Protocole OPAQUE (mots de passe ne quittant jamais le client), WebAuthn avec extension PRF (Passkeys), et TOTP.

---

## 2. Matrice des Menaces et Mitigations (Évaluation Honnête)

| Vecteur d'Attaque | Statut | Description & Mitigation dans Lureva |
| :--- | :--- | :--- |
| **Fuite de Base de Données (Data Breach)** | 🟢 Protégé | PostgreSQL ne stocke que des blobs chiffrés et des clés enveloppées. Les mots de passe ne sont **pas stockés**, même sous forme de hash (grâce au protocole OPAQUE). Les données utilisateurs restent illisibles. |
| **Vol de Session (Session Hijacking / Cookie Theft)** | 🟢 Protégé | Les cookies JWT sont `HttpOnly`. Même si un attaquant vole le cookie, le serveur rejettera ses requêtes. **Pourquoi ?** Le JWT est lié cryptographiquement à l'appareil via **DPoP**. Sans la clé privée stockée en mémoire volatile sur l'appareil de l'utilisateur, le cookie est inutile. |
| **Attaques par Ordinateur Quantique** | 🟢 Protégé | L'implémentation de la librairie `@noble/post-quantum` (ML-KEM) garantit que les données interceptées aujourd'hui ne pourront pas être déchiffrées par les ordinateurs quantiques de demain ("*Harvest Now, Decrypt Later*"). |
| **Keyloggers & Credential Stuffing** | 🟢 Protégé | 1. Les Passkeys (WebAuthn) sont immunisés contre le phishing et les keyloggers.<br>2. Pour le login par mot de passe, le **TOTP (2FA)** bloque l'utilisation d'un mot de passe volé. |
| **Attaque CSRF (Cross-Site Request Forgery)** | 🟢 Protégé | Bloqué de manière inhérente par le mécanisme **DPoP** (voir explication détaillée ci-dessous), couplé aux attributs `SameSite=Lax/None` des cookies. |
| **Espionnage d'État / Lois Extraterritoriales** | 🟢 Protégé | Hébergement sur un Cloud Souverain Français. Contrairement aux hébergeurs US (AWS, Google, Cloudflare), aucune agence étrangère ne peut exiger la remise des métadonnées des utilisateurs via le CLOUD Act. |
| **Déni de Service (DDoS) & Brute Force** | 🟡 Partiel | Le moteur d'exécution **Bun** est extrêmement performant et le limiteur de requêtes (Rate Limiting) en mémoire est ultra-efficace sur une instance unique. *Cependant*, l'infrastructure nécessite un bouclier Anti-DDoS L4/L7 fourni par l'hébergeur (ex: OVH Arbor) pour encaisser les attaques volumétriques. |
| **Administrateur Malveillant (Insider Threat)** | 🟡 Partiel | Un admin base de données ne peut pas lire les données (Zero-Knowledge). *Cependant*, il peut supprimer les données (déni de service) ou servir un code JavaScript malveillant pour voler les clés à la volée. |
| **Attaque XSS (Cross-Site Scripting)** | 🟡 Partiel | **Le plus grand risque des applications E2EE web.** Les clés privées sont générées avec `extractable: false` (impossible à exfiltrer). *Cependant*, un script injecté pourrait utiliser l'API `crypto.subtle` locale pour déchiffrer les données *tant que l'utilisateur est connecté et actif*. Une politique CSP (Content Security Policy) stricte est requise. |

---

## 3. Focus : Pourquoi les attaques CSRF sont inefficaces dans cette suite ?

De base, le protocole **DPoP (Demonstrating Proof-of-Possession)** n'a pas été inventé explicitement pour contrer le CSRF, mais pour empêcher le vol de tokens. Cependant, de par son fonctionnement cryptographique, **il devient le mécanisme anti-CSRF le plus puissant qui existe.**

### Comment fonctionne une attaque CSRF classique ?
1. Vous êtes connecté à `example.com`. Votre navigateur possède un cookie de session.
2. Un attaquant vous fait visiter `site-malveillant.com`.
3. Ce site envoie une requête invisible en arrière-plan vers `api.example.com/mail/delete/all`.
4. Le navigateur, voyant que la requête va vers `example.com`, **joint automatiquement le cookie de session**.
5. Le serveur voit un cookie valide et exécute l'action.

### Pourquoi le DPoP détruit cette attaque dans Lureva :
Dans le backend, Lureva exige trois éléments pour les requêtes modifiant l'état (POST, DELETE, etc.) :
1. Le Cookie JWT.
2. L'en-tête `x-dpop-signature`.
3. L'en-tête `x-dpop-timestamp`.

Sur `site-malveillant.com`, l'attaquant peut forcer votre navigateur à envoyer le Cookie. **Mais il ne peut pas forger l'en-tête `x-dpop-signature`**.

Pour forger cette signature, le script de l'attaquant aurait besoin de la **Clé Privée DPoP** de l'utilisateur (générée localement lors du login). 
Or, cette clé privée :
1. N'est connue que de l'application React hébergée sur `example.com`.
2. Est stockée en mémoire avec le flag `extractable: false` (le navigateur interdit même à Lureva de la lire, elle peut juste être *utilisée* pour signer par le domaine légitime).

**Résultat :** Le site malveillant envoie une requête avec un cookie valide, mais sans la signature DPoP. Le serveur Bun/PostgreSQL la rejette immédiatement avec un code `403 Forbidden: Cryptographic signature mismatch`. Le CSRF est mathématiquement impossible.

---

## 4. Comparaison Concurrentielle (SaaS E2EE)

Voici comment Lureva se positionne face aux acteurs majeurs du marché en juin 2026 :

| Fonctionnalité | 🚀 **Lureva** | 🛡️ **Proton** (Mail/Drive) | 📝 **Notion** | 🏢 **Google Workspace** |
| :--- | :--- | :--- | :--- | :--- |
| **Souveraineté (Anti-CLOUD Act)** | ✅ **France / UE** (Aucun accès US) | ✅ Suisse | ❌ US (AWS) | ❌ US |
| **Chiffrement de bout en bout (E2EE)** | ✅ Oui (Défaut) | ✅ Oui | ❌ Non (En transit/repos seulement) | 🟡 Optionnel (Client-side encryption payant) |
| **Support Post-Quantique (PQC)** | ✅ **Défaut** (ML-KEM 768) | 🟡 Optionnel (Désactivé par défaut) | ❌ Non | 🟡 En déploiement interne |
| **Authentification Zero-Knowledge** | ✅ **OPAQUE** (Mot de passe caché) | 🟡 SRP (Vieux standard) | ❌ Standard | ❌ Standard |
| **Passwordless natif (Passkeys)**| ✅ Oui (Via WebAuthn PRF) | ❌ Mdp requis pour le déchiffrement | ✅ Oui (Mais pas E2EE) | ✅ Oui (Mais pas E2EE) |
| **Protection contre le vol de session**| ✅ Oui (DPoP) | 🟡 Sessions révocables | 🟡 Sessions révocables | ✅ Détection d'anomalies IA |
| **Protection des Bases de Données** | ✅ PostgreSQL (Intégrité JSONB) | ✅ Base centralisée SQL | ✅ Centralisée | ✅ Distribuée mondiale |
| **Récupération de compte d'Entreprise** | ✅ Key Escrow asymétrique | ✅ Oui | N/A | ✅ Accès Super Admin total |

### Conclusion
L'architecture actuelle de la suite Lureva est d'une robustesse exceptionnelle. En combinant la **souveraineté des données française**, les performances brutes du moteur **Bun + PostgreSQL**, et les standards cryptographiques de demain imposés par défaut (**DPoP, OPAQUE, PRF, Post-Quantique**), Lureva se positionne techniquement au-dessus des géants actuels de la confidentialité comme Proton, qui peinent encore à imposer le post-quantique et les Passkeys E2EE par défaut en raison du poids de leur infrastructure legacy.
