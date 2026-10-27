# Base de données MbScanner (schéma v2)

Cette branche contient la base d'équipements utilisée par MbScanner pour identifier les appareils Modbus
et compléter le certificat de communication.

| Fichier | Rôle |
|---|---|
| `database.db` | Base SQLite téléchargée par l'application |
| `version.txt` | Version de la base. L'application télécharge `database.db` dès que cette valeur diffère de sa version locale |

> **Branches** : `main` héberge l'ancien schéma (v1), lu par la version de MbScanner déployée en production.
> Elle ne doit pas être modifiée. Cette branche `v2` est lue par la nouvelle version de l'application.

## Modifier la base

1. Modifier la base de travail d'un poste (`%APPDATA%\MbScanner\databases\database.db`), par exemple avec DB Browser for SQLite.
2. Publier immédiatement avec `python tools/publish_database.py <clone de ce dépôt>` (dépôt MbScanner). Le script
   vide la table `certificate`, compacte la base, vérifie le schéma et incrémente `version.txt`.
3. Relire le résumé affiché, puis commiter et pousser sur `v2`.

Ce dépôt est **public** : ne jamais y pousser une base de travail sans passer par le script.
Une modification locale non publiée est écrasée à la prochaine mise à jour de la base sur ce poste.

## Principe de l'identification

Pour chaque adresse esclave, l'application essaie dans l'ordre :

1. **Identification générique (fonction 43/0E)** : elle lit le code produit (objet `0x01`) et cherche l'équipement
   dont `product_code`, ou à défaut `reference`, est égal à ce code.
2. **Sondes de marque** (`brand_probe`) : elles déterminent la marque de l'appareil.
3. **Signatures** (`equipment_signature`) des équipements de la marque trouvée : elles déterminent le modèle.
   Si aucune sonde ne répond, l'application essaie d'abord les marques sans sonde (ABB, iSMA…), puis toutes les signatures.

Une fois le modèle connu, l'application lit le numéro de série (`equipment_serial`) et, pour les compteurs à mesure
indirecte, les réglages des TC (`undirect_meters`).

---

## Codes communs

Les tables `brand_probe` et `equipment_signature` partagent les colonnes suivantes.

### `method` : type de lecture

| Valeur | Lecture | `address` | `length` | `var_type` |
|---|---|---|---|---|
| `1` | Registres de maintien (fonction 03) | Premier registre (adresse 0-based) | Nombre de registres | Obligatoire |
| `2` | Identification d'équipement (43/0E) | *Read Device ID code* : `0` ou `1` (catégorie basique, comme dans les lignes existantes) | Index de l'objet : `0` = fabricant, `1` = code produit, `2` = version | Vide (`NULL`) |

### `var_type` : décodage des registres (`method = 1`)

| Valeur | Décodage | Usage |
|---|---|---|
| `string` | Texte ASCII, 2 caractères par registre, les `\0` sont retirés | Noms, références |
| `uint_be` | Entier non signé 16 bits, big-endian | **À utiliser pour les nouvelles entrées** : la valeur attendue est celle de la doc constructeur |
| `int_be` | Entier signé 16 bits, big-endian | Idem |
| `float_be` | Flottant 32 bits, big-endian | Idem |
| `uint` / `int` / `float` | Comme ci-dessus, mais **octets de chaque registre inversés** | Historique : entrées héritées de l'ancien moteur, ne pas utiliser pour une nouvelle entrée |

### `operator` : comparaison entre la valeur lue et `expected`

La valeur lue est toujours convertie en texte avant la comparaison.

| Valeur | Opérateur | Remarque |
|---|---|---|
| `1` | `<` | Comparaison de **textes** (ordre alphabétique) |
| `2` | `<=` | Idem |
| `3` | `=` | Égalité stricte (casse comprise) |
| `4` | `>=` | Comparaison de textes |
| `5` | `>` | Comparaison de textes |
| `6` | Expression régulière | **Le plus utilisé.** Insensible à la casse, ancrée au début. Utiliser `^...$` pour une correspondance exacte, `^(A\|B)$` pour plusieurs valeurs |
| `7` | Égalité insensible à la casse | |

---

## Tables

### `equipment` : catalogue des équipements

Une ligne par modèle reconnu. C'est la table centrale : les autres y font référence par `equipment_id`.

| Colonne | Type | Obligatoire | Description |
|---|---|---|---|
| `id` | entier | auto | Identifiant |
| `reference` | texte | oui, unique | Référence catalogue (ex. `METSEPM8240`). Affichée dans les résultats et sur le certificat. Elle sert aussi de code produit 43/0E quand `product_code` est vide |
| `name` | texte | oui | Désignation lisible (ex. `PM8240`). Affichée dans la colonne Équipement du certificat |
| `brand` | texte | oui | Marque. Doit être **écrite exactement** comme dans `brand_probe.brand` (ex. `Schneider Electric`), sinon les signatures du modèle ne seront essayées qu'en dernier recours. Pour les compteurs indirects, elle choisit aussi la table de traduction des systèmes de mesure |
| `product_code` | texte | non | Code renvoyé en 43/0E (objet `0x01`), **uniquement s'il diffère de `reference`**. Ex. `reference = METSEPM5560`, l'appareil répond `PM5560` → `product_code = PM5560`. Laisser vide si le code est égal à la référence |

Un message `answers 43/0E with unknown product code 'XXX'` dans les journaux indique qu'il suffit de renseigner
`XXX` dans `product_code` (ou `reference`) de l'équipement concerné.

### `brand_probe` : sondes de marque

Lectures qui déterminent la marque d'un appareil. Elles sont essayées par ordre de `priority` : chaque sonde qui
correspond ajoute sa marque, puis l'application ne teste que les signatures de ces marques.

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `brand` | texte | Marque reconnue. Doit correspondre à `equipment.brand` |
| `method` | entier | Type de lecture (voir [codes communs](#method--type-de-lecture)) |
| `address` | entier | Registre, ou *Read Device ID code* en 43/0E |
| `length` | entier | Nombre de registres, ou index d'objet en 43/0E |
| `var_type` | texte | Décodage (vide en 43/0E) |
| `operator` | entier | Comparaison (en pratique `6`, expression régulière) |
| `expected` | texte | Valeur attendue (ex. `^Schneider Electric$`) |
| `priority` | entier | Ordre d'essai, du plus petit au plus grand. Mettre en premier les sondes rapides et les marques fréquentes |

Une même marque peut avoir plusieurs sondes (ex. Schneider : une en 43/0E, une par registre pour les appareils sans 43/0E).
Une marque sans sonde reste identifiable par ses signatures seules.

### `equipment_signature` : signatures de modèle

Conditions qui identifient un modèle précis.

- Les lignes d'un même équipement **et** d'une même `variant` doivent **toutes** correspondre (ET).
- Il suffit qu'**une** variante corresponde pour identifier l'équipement (OU).
- Les variantes comptant le plus de conditions sont essayées en premier, car ce sont les plus précises.

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `equipment_id` | entier | Équipement identifié (`equipment.id`) |
| `variant` | entier | Numéro de variante (`0` par défaut). Utiliser `1`, `2`… pour d'autres manières de reconnaître le même modèle (ex. autre firmware) |
| `rank` | entier | Ordre des conditions au sein de la variante (`0`, `1`…). Mettre d'abord la condition la plus discriminante |
| `method` | entier | Type de lecture |
| `address` | entier | Registre, ou *Read Device ID code* |
| `length` | entier | Nombre de registres, ou index d'objet |
| `var_type` | texte | Décodage (vide en 43/0E) |
| `operator` | entier | Comparaison |
| `expected` | texte | Valeur attendue |

Exemple : Micrologic 5.2 E, identifié en deux conditions :

| equipment_id | variant | rank | method | address | length | var_type | operator | expected |
|---|---|---|---|---|---|---|---|---|
| 161 | 0 | 0 | 2 | 0 | 1 | | 6 | `^LV434000$` |
| 161 | 0 | 1 | 1 | 29999 | 4 | string | 6 | `^LV429095$` |

Un équipement Schneider dont le code 43/0E est égal à sa référence (ou à son `product_code`) n'a besoin
d'**aucune** signature.

### `equipment_serial` : emplacement du numéro de série

Facultatif, au plus une ligne par équipement. Si l'emplacement est déclaré, le numéro de série est lu après
l'identification et imprimé sous le modèle sur le certificat. Une lecture qui échoue n'empêche pas le test.

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `equipment_id` | entier | Équipement concerné |
| `address` | entier | Premier registre (fonction 03, adresse 0-based) |
| `length` | entier | Nombre de registres à lire |
| `encoding` | texte | `string` : 2 caractères ASCII par registre, octet de poids fort d'abord. `low_byte` : 1 caractère par registre, dans l'octet de poids faible (ex. Carlo Gavazzi EM111) |

### `undirect_meters` : compteurs à mesure indirecte (TC)

Lectures des réglages des transformateurs de courant, imprimés dans la section « Compteurs à mesure indirecte »
du certificat. Un équipement présent dans cette table est considéré comme un compteur indirect. Il faut **une ligne
par valeur** à lire.

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `equipment_id` | entier | Compteur concerné |
| `type` | entier | `1` = lecture de registres (fonction 03). `2` = identification 43/0E (rarement utile) |
| `data` | texte (JSON) | Où et comment lire, voir ci-dessous |
| `data_type` | entier | Valeur représentée, voir ci-dessous |

**`data`** est un objet JSON. Il doit être valide : guillemets doubles, pas de virgule finale.

```json
{"register": 56000, "length": 2, "type": "dint", "endianness": 1}
```

| Clé | Description |
|---|---|
| `register` | Premier registre (adresse 0-based) |
| `length` | Nombre de registres : `1` pour `int`, `2` pour `dint` |
| `type` | `int` : 1 registre, valeur brute non signée. `dint` : entier signé 32 bits sur 2 registres, mot de poids fort d'abord. `string` : texte |
| `endianness` | Utilisé seulement pour `string`. Mettre `1` |

**`data_type`** :

| Valeur | Signification | Affichage sur le certificat |
|---|---|---|
| `1` | Système de mesure (code) | Traduit selon la marque (section `[POWER_SYSTEM]` de la configuration de l'application) |
| `2` | Nombre de TC | Valeur brute |
| `3` | Rapport TC (primaire / 5) | `valeur × 5` suivi de `A/5A` |
| `4` | Courant primaire des TC | `primaire A / secondaire A` |
| `5` | Courant secondaire des TC | Complète le `4`. Sans ligne `5`, le secondaire vaut 5 A |

Une lecture refusée par l'appareil s'affiche « Non lu » pour la valeur concernée, sans bloquer les autres.

> Dans les bases antérieures à la 2.0.3, la colonne `data` était déclarée `INTEGER`, et DB Browser remplaçait
> le texte saisi par `0`. Si c'est le cas, passer la colonne en `TEXT` avant de modifier.

### `operator` : opérateurs

Liste des personnes qui peuvent réaliser un test. Le choix d'un opérateur est obligatoire pour lancer un test, et son
nom est imprimé sur le certificat (« Testé le … à … par … »).

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `name` | texte, unique | Nom affiché dans la liste déroulante (triée par ordre alphabétique) et sur le certificat |

### `certificate` : historique local des certificats

**Toujours vide dans ce dépôt.** Chaque poste y enregistre les certificats qu'il génère, avec des données client.
À chaque mise à jour, l'application recopie l'historique local dans la base téléchargée.

| Colonne | Type | Description |
|---|---|---|
| `id` | entier | Identifiant |
| `product` | texte (JSON) | Produit / affaire testé (numéro de série, client, affaire, références…) |
| `status` | entier | `0` = pas encore transmis au serveur, `1` = transmis |
| `filename` | texte | Chemin du PDF généré |
| `date` | entier | Date de génération (timestamp Unix) |
