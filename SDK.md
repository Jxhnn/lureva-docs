
# Guide d'intégration : SDK Lureva API (E2EE)

*(Ce document est destiné aux développeurs externes ou aux clients Enterprise souhaitant automatiser des actions sur Lureva)*

## 1. Introduction à l'API zero-knowledge
Bienvenue dans la documentation développeur de Lureva.
La majorité des API SaaS (comme Google Workspace ou Notion) vous permettent d'envoyer un texte en clair (ex: `{"title": "Mon Document"}`) avec un simple Token Bearer.

**Chez Lureva, cela est impossible.** Notre architecture Zero-Knowledge nous interdit de recevoir vos données en clair.
Pour interagir avec Lureva par API, vous devez utiliser notre **SDK Client (`@lureva/vault-client-sdk`)** qui se chargera de la cryptographie locale avant de parler à nos serveurs.

## 2. Installation du SDK
Le SDK est conçu pour fonctionner dans n'importe quel environnement JavaScript moderne (Node.js, Deno, Bun, ou Navigateur).

```bash
npm install @lureva/vault-client-sdk
```

## 3. Authentification (machine-to-machine)
Pour qu'un script puisse agir en votre nom, il doit dériver votre clé maîtresse (Master Account Key).

```typescript
import { LurevaClient } from '@lureva/vault-client-sdk';

// 1. Initialisation du client
const lureva = new LurevaClient({
  environment: 'production' // cible l'api officielle
});

// 2. Authentification avec protocole OPAQUE
// Le mot de passe ne quitte jamais votre machine.
// Il est utilisé localement pour déchiffrer votre coffre-fort.
await lureva.auth.login({
  email: 'robot@mon-entreprise.com',
  password: 'MonSuperMotDePasseMachine123!'
});

console.log("Connecté et coffre-fort déchiffré localement !");
```

## 4. Manipuler le drive (fichiers chiffrés)
Le SDK gère automatiquement le chiffrement des flux (Streams) et la génération des clés AES uniques pour chaque fichier.

### Uploader un fichier
```typescript
import { createReadStream } from 'fs';

const fileStream = createReadStream('./rapport_confidentiel.pdf');

// Le SDK va :
// 1. Générer une clé AES aléatoire pour ce fichier.
// 2. Chiffrer le fichier à la volée.
// 3. Demander une URL pré-signée au serveur Lureva.
// 4. Uploader le blob chiffré sur l'Object Storage.
// 5. Chiffrer le nom du fichier et la clé AES avec votre clé publique et l'envoyer à la DB.
const uploadedFile = await lureva.drive.uploadFile({
  name: 'Rapport confidentiel.pdf',
  stream: fileStream,
  folderId: null // null = Racine du Drive
});

console.log(`Fichier chiffré et stocké sous l'ID : ${uploadedFile.id}`);
```

### Télécharger et déchiffrer un fichier
```typescript
import { createWriteStream } from 'fs';

const outStream = createWriteStream('./downloaded_rapport.pdf');

// Le SDK télécharge le blob, récupère la clé AES enveloppée depuis votre profil,
// la déchiffre en mémoire, et déchiffre le fichier à la volée.
await lureva.drive.downloadFile({
  fileId: 'uuid-du-fichier',
  destination: outStream
});
```

## 5. Manipuler les listes (bases de données E2EE)
Les "Listes" de Lureva fonctionnent sur un modèle d'Event Sourcing (CRDT). Pour ajouter une ligne de base de données, vous ajoutez un "Événement" chiffré.

```typescript
// 1. Récupérer l'accès à la Liste
const list = await lureva.lists.get('uuid-de-la-liste');

// 2. Ajouter un nouvel item (Ligne de tableau)
// Les clés des colonnes et leurs valeurs sont chiffrées localement.
await list.items.create({
  content: {
    "Statut": "En cours",
    "Client": "Acme Corp",
    "Montant": 50000
  }
});

console.log("Donnée ajoutée et chiffrée avec succès !");
```

## 6. Sécurité DPoP (Demonstrating Proof-of-Possession)
En interne, le SDK Lureva génère une paire de clés éphémère lors de la méthode `login()`. Toutes les requêtes HTTP suivantes sont signées cryptographiquement.
Si vous tentez d'intercepter le Token réseau de votre propre script pour le rejouer dans un outil comme *Postman* ou *cURL*, **la requête sera rejetée avec une erreur 403**. C'est le comportement attendu qui garantit que vos sessions ne peuvent pas être volées.
