
# Lureva : conformité, vie privée et RGPD

*(Ce document est destiné aux délégués à la protection des données (DPO), aux équipes juridiques et aux utilisateurs soucieux de leur vie privée).*

## 1. Notre engagement : le zero-knowledge
Chez Lureva, nous croyons que la confidentialité est un droit fondamental. Contrairement aux plateformes cloud traditionnelles, notre architecture garantit que **nous n'avons aucun moyen technique de lire vos fichiers, vos e-mails ou vos bases de données**. Vos données sont chiffrées sur votre appareil *avant* d'être envoyées sur nos serveurs.

## 2. Cartographie des données (data mapping)

Pour assurer le fonctionnement du service et la facturation, une stricte séparation est appliquée entre les données "en clair" (nécessaires au routage) et les données "chiffrées" (votre contenu).

### 🔴 Ce que nous NE POUVONS PAS voir (Chiffrement E2EE) :
*   Le contenu de vos e-mails et pièces jointes.
*   Le nom et le contenu de vos fichiers / dossiers Drive.
*   Le nom, la structure et les données à l'intérieur de vos listes.
*   Votre mot de passe (protégé par le protocole OPAQUE, il ne quitte jamais votre appareil).
*   Vos clés privées (chiffrées par votre mot de passe ou l'extension PRF de vos passkeys).

### 🟢 Ce que nous POUVONS voir (métadonnées de routage) :
*   Votre adresse e-mail de connexion.
*   Vos informations de facturation (Plan choisi, statut de l'abonnement).
*   L'espace de stockage utilisé (en octets).
*   Les adresses e-mail d'expédition et de destination (nécessaire pour le protocole SMTP / routage des e-mails).
*   Les adresses IP de connexion (conservées temporairement pour des raisons de sécurité et de prévention des abus).

## 3. Souveraineté et hébergement (Anti-CLOUD Act)
Lureva est une entreprise européenne. L'intégralité de notre infrastructure (serveurs de calcul et stockage S3) est hébergée en **France** chez des fournisseurs cloud européens souverains (ex: Scaleway / OVHcloud).

**Pourquoi c'est important ?**
Vos données ne sont pas soumises aux lois extraterritoriales américaines telles que le *CLOUD Act* ou le *FISA*. Aucune agence gouvernementale étrangère ne peut nous contraindre à leur fournir un accès. De plus, notre architecture zero-knowledge rend toute réquisition de contenu techniquement inopérante.

## 4. Exercice de vos droits (RGPD)
Conformément au Règlement Général sur la Protection des Données (RGPD), vous disposez d'un contrôle total sur votre compte :

*   **Droit à l'effacement (droit à l'oubli) :** la suppression de votre compte depuis vos paramètres entraîne la destruction *immédiate et irréversible* de vos métadonnées en base de données, ainsi que la suppression physique de vos blobs chiffrés sur nos serveurs de stockage.
*   **Droit à la portabilité :** vous pouvez télécharger l'intégralité de vos fichiers depuis le Drive Lureva et exporter vos clés cryptographiques à tout moment.
*   **Limitation technique (perte de mot de passe) :** dans le cadre des comptes personnels, si vous perdez votre mot de passe, vos Passkeys ET votre phrase de récupération, **vos données seront perdues à tout jamais**. Lureva ne possède aucune porte dérobée (*backdoor*) pour restaurer vos données.

## 5. Sous-traitants (sub-processors)
Lureva limite au strict minimum le partage de métadonnées avec des tiers :
*   **Stripe (Irlande / US) :** Utilisé uniquement pour le traitement sécurisé des paiements. Ne reçoit que votre adresse e-mail et vos données de carte bancaire.
*   **[OVH] (France) :** hébergement physique des serveurs. N'a accès à aucune donnée en clair (disques chiffrés + base de données ne contenant que des blobs AES-GCM).
euvent pas être volées.
