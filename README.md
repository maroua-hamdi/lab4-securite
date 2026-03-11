# Rapport de Lab — Analyse statique d'APK Android
### OWASP UnCrackable Android App — Level 1

---

## Informations générales

- **APK analysé :** `UnCrackable-Level1.apk` (OWASP MSTG)
- **Hash SHA-256 :** `1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21`
- **Outils utilisés :** PowerShell, JADX GUI, dex2jar, JD-GUI
- **Environnement :** Windows 11 — `D:\APK-Analysis`

---

## Task 1 — Préparer le workspace et vérifier l'APK

![Task 1 - Instructions](1773265104214_image.png)

**Création du dossier et vérification des magic bytes :**

On vérifie que l'APK commence bien par `50 4B` (`PK`) — signature d'une archive ZIP valide. Tout APK Android est en réalité un fichier ZIP, donc ces octets doivent toujours être présents.

![Task 1 - Magic bytes](1773265279359_image.png)

**Listing du contenu de l'APK :**

Sans l'extraire, on liste les fichiers internes. On retrouve les composants attendus : `AndroidManifest.xml`, `classes.dex` (le code compilé), les certificats de signature dans `META-INF/`, et les ressources graphiques.

![Task 1 - Contenu ZIP](1773265295354_image.png)

**Calcul du hash SHA-256 :**

Le hash sert à identifier de façon unique le fichier analysé et garantir qu'il n'a pas été modifié. Il sera mentionné dans le rapport final pour la traçabilité.

![Task 1 - Hash SHA-256](1773265311536_image.png)

---

## Task 2 — Extraire/obtenir l'APK

![Task 2 - Instructions](1773265117769_image.png)

L'APK `UnCrackable-Level1.apk` a été utilisé directement (Option A — APK fourni). Placé dans `D:\APK-Analysis` sans manipulation supplémentaire.

---

## Task 3 — Analyse avec JADX GUI

![Task 3 - Instructions](1773265132241_image.png)

**Exploration dans JADX GUI :**

JADX décompile le bytecode `classes.dex` en Java lisible. On explore la structure dans le panneau gauche et on identifie les packages :
- `sg.vantagepoint.uncrackable1` → package principal avec `MainActivity`
- `sg.vantagepoint.a` → contient la logique de vérification du secret

Dans la classe `a`, on trouve deux éléments critiques codés en dur :
- Une **clé AES en hexadécimal** : `8d127684cbc37c17616d806cf50473cc`
- Un **secret chiffré en Base64** : `5UJiFctbmgbDoLXmpL12mknо8HT4Lv8dlat8FxR2GOc=`

Ces valeurs hardcodées permettent de reconstituer le secret sans exécuter l'application.

![Task 3 - JADX GUI, classe a](1773265346111_image.png)

---

## Task 4 — Recherche de chaînes sensibles

![Task 4 - Instructions](1773265148258_image.png)

**Méthode `verify` dans `MainActivity` :**

Quand l'utilisateur appuie sur le bouton, cette méthode récupère la saisie et appelle `a.a(string)`. Si le retour est `true`, l'app affiche "Success!". Toute la logique est côté client — il suffit de comprendre ce que `a.a()` calcule pour trouver le secret.

![Task 4 - Méthode verify](1773265366615_image.png)

**Méthode `b` — conversion hex vers bytes :**

Transforme la chaîne hexadécimale `8d127684cbc37c17616d806cf50473cc` en 16 octets, qui constituent la clé AES-128 utilisée pour le déchiffrement.

![Task 4 - Méthode b hex](1773265457497_image.png)

---

## Task 5 — Convertir DEX vers JAR avec dex2jar

![Task 5 - Instructions](1773265162768_image.png)

On extrait `classes.dex` de l'APK puis on le convertit en `app.jar` avec dex2jar. Nécessaire pour ouvrir le code dans JD-GUI, qui ne lit que le format JAR/CLASS (Java) et pas le format DEX (Android).

---

## Task 6 — Comparaison JADX vs JD-GUI

![Task 6 - Instructions](1773265177211_image.png)

On ouvre `app.jar` dans JD-GUI et on compare avec JADX sur la même classe. Différences notables :
- JADX donne accès aux ressources et au manifest, JD-GUI non
- JADX gère mieux Kotlin et les annotations Android
- JD-GUI conserve les noms obfusqués tels quels, JADX tente de les reconstruire

Conclusion : JADX est plus complet pour l'analyse Android. JD-GUI reste utile en vérification secondaire.

---

## Task 7 — Rédiger le mini-rapport

![Task 7 - Instructions](1773265204372_image.png)

Vulnérabilités identifiées :

| # | Constat | Sévérité | Localisation |
|---|---------|----------|--------------|
| 1 | Clé AES hardcodée | Élevée | `sg.vantagepoint.a.a` |
| 2 | Secret chiffré hardcodé | Élevée | `sg.vantagepoint.a.a` |
| 3 | Vérification entièrement côté client | Élevée | `MainActivity.verify()` |
| 4 | Obfuscation minimale | Moyenne | Global |

---

## Task 8 — Nettoyage

![Task 8 - Instructions](1773265217360_image.png)

- Rapport vérifié (aucune info sensible réelle incluse)
- Fichiers organisés dans `results/`
- Dossier temporaire `dex_out/` supprimé

---

## Résultat final — Secret découvert

Grâce à l'analyse statique, on reconstruit la chaîne complète :
1. La clé AES hex est convertie en bytes via la méthode `b()`
2. Le secret Base64 est décodé
3. Le déchiffrement AES produit le secret en clair
4. La comparaison dans `verify()` confirme la réponse

**Validation dans l'émulateur :**

![Résultat final](1773265255483_image.png)

> **Secret : `I want to believe`**
>
> Le chiffrement AES ne protège pas ici car la clé est stockée au même endroit que les données chiffrées — c'est comme fermer un coffre et laisser la clé dessus.
