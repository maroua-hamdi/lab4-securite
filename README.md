# Rapport de Lab — Analyse statique d'APK Android
### OWASP UnCrackable Android App — Level 1

---

## Informations générales

- **APK analysé :** `UnCrackable-Level1.apk` (OWASP MSTG)
- **Hash SHA-256 :** `1DA8BF57D266109F9A07C01BF7111A1975CE01F190B9D914BCD3AE3DBEF96F21`
- **Outils utilisés :** PowerShell, JADX GUI, dex2jar, JD-GUI


---

## Task 1 — Préparer le workspace et vérifier l'APK

**Création du dossier et vérification des magic bytes :**

On vérifie que l'APK commence bien par `50 4B` (`PK`) — signature d'une archive ZIP valide. Tout APK Android est en réalité un fichier ZIP, donc ces octets doivent toujours être présents.

<img width="1225" height="662" alt="image" src="https://github.com/user-attachments/assets/0db9cc6d-f755-40aa-81c9-b379db730811" />


**Listing du contenu de l'APK :**

Sans l'extraire, on liste les fichiers internes. On retrouve les composants attendus : `AndroidManifest.xml`, `classes.dex` (le code compilé), les certificats de signature dans `META-INF/`, et les ressources graphiques.

<img width="1201" height="343" alt="image" src="https://github.com/user-attachments/assets/7742ea7a-bb85-48d0-a71f-a3dc048798a5" />


**Calcul du hash SHA-256 :**

Le hash sert à identifier de façon unique le fichier analysé et garantir qu'il n'a pas été modifié. Il sera mentionné dans le rapport final pour la traçabilité.

<img width="1187" height="179" alt="image" src="https://github.com/user-attachments/assets/ba4a6591-7fc6-4f58-ad06-7b7f00128cfe" />


---

## Task 2 — Extraire/obtenir l'APK

L'APK `UnCrackable-Level1.apk` a été utilisé directement (Option A — APK fourni). Placé dans `D:\APK-Analysis` sans manipulation supplémentaire.

---

## Task 3 — Analyse avec JADX GUI

**Exploration dans JADX GUI :**

JADX décompile le bytecode `classes.dex` en Java lisible. On explore la structure dans le panneau gauche et on identifie les packages :
- `sg.vantagepoint.uncrackable1` → package principal avec `MainActivity`
- `sg.vantagepoint.a` → contient la logique de vérification du secret

Dans la classe `a`, on trouve deux éléments critiques codés en dur :
- Une **clé AES en hexadécimal** : `8d127684cbc37c17616d806cf50473cc`
- Un **secret chiffré en Base64** : `5UJiFctbmgbDoLXmpL12mknо8HT4Lv8dlat8FxR2GOc=`

Ces valeurs hardcodées permettent de reconstituer le secret sans exécuter l'application.

<img width="1217" height="491" alt="image" src="https://github.com/user-attachments/assets/f91c3060-2010-4341-911a-cf4e2c41c576" />


---

## Task 4 — Recherche de chaînes sensibles

**Méthode `verify` dans `MainActivity` :**

Quand l'utilisateur appuie sur le bouton, cette méthode récupère la saisie et appelle `a.a(string)`. Si le retour est `true`, l'app affiche "Success!". Toute la logique est côté client — il suffit de comprendre ce que `a.a()` calcule pour trouver le secret.

<img width="1223" height="374" alt="image" src="https://github.com/user-attachments/assets/f874ae55-965b-498d-ab7b-e2b4d98224d2" />


**Méthode `b` — conversion hex vers bytes :**

Transforme la chaîne hexadécimale `8d127684cbc37c17616d806cf50473cc` en 16 octets, qui constituent la clé AES-128 utilisée pour le déchiffrement.

<img width="1220" height="195" alt="image" src="https://github.com/user-attachments/assets/ee69da8f-38bc-426d-a154-8cf7907f971f" />

---

## Task 5 — Convertir DEX vers JAR avec dex2jar

On extrait `classes.dex` de l'APK puis on le convertit en `app.jar` avec dex2jar. Nécessaire pour ouvrir le code dans JD-GUI, qui ne lit que le format JAR/CLASS (Java) et pas le format DEX (Android).

---

## Task 6 — Comparaison JADX vs JD-GUI

On ouvre `app.jar` dans JD-GUI et on compare avec JADX sur la même classe. Différences notables :
- JADX donne accès aux ressources et au manifest, JD-GUI non
- JADX gère mieux Kotlin et les annotations Android
- JD-GUI conserve les noms obfusqués tels quels, JADX tente de les reconstruire

Conclusion : JADX est plus complet pour l'analyse Android. JD-GUI reste utile en vérification secondaire.

---

## Task 7 — Rédiger le mini-rapport

Vulnérabilités identifiées :

| # | Constat | Sévérité | Localisation |
|---|---------|----------|--------------|
| 1 | Clé AES hardcodée | Élevée | `sg.vantagepoint.a.a` |
| 2 | Secret chiffré hardcodé | Élevée | `sg.vantagepoint.a.a` |
| 3 | Vérification entièrement côté client | Élevée | `MainActivity.verify()` |
| 4 | Obfuscation minimale | Moyenne | Global |

---

## Task 8 — Nettoyage

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

<img width="632" height="519" alt="image" src="https://github.com/user-attachments/assets/20f8cba0-a1a2-408f-8174-6df7800338b0" />


> **Secret : `I want to believe`**
>
> Le chiffrement AES ne protège pas ici car la clé est stockée au même endroit que les données chiffrées — c'est comme fermer un coffre et laisser la clé dessus.
