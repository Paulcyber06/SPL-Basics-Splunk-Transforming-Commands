# Épisode 3 — Splunk : Analyse Comportementale d'une IP Suspecte

> Troisième incident dans la série d'attaques contre **Buttercup Games** — l'IP suspecte n'a pas trouvé `/passwords.pdf` mais utilise activement le site. Le SOC approfondit l'investigation avec les commandes transforming **stats**, **top** et **timechart**.

---

## 📚 Table des matières

- [1. Contexte](#1-contexte)
- [2. Vérification des accès réussis](#2-vérification-des-accès-réussis)
- [3. Analyse avec stats — pages les plus accédées](#3-analyse-avec-stats--pages-les-plus-accédées)
- [4. Analyse avec top — confirmation](#4-analyse-avec-top--confirmation)
- [5. Analyse temporelle avec timechart](#5-analyse-temporelle-avec-timechart)
- [6. Vue détaillée par status code](#6-vue-détaillée-par-status-code)
- [7. Conclusion](#7-conclusion)
- [8. Réponse opérationnelle SOC](#8-réponse-opérationnelle-soc)

---

## 1. Contexte

Suite à la détection de l'activité de reconnaissance dans l'Épisode 1, le SOC approfondit l'investigation sur l'IP suspecte `87.194.216.51`, en vérifieant les requêtes 200 (Status OK). 

L'objectif est de comprendre son comportement complet — pas uniquement les erreurs 404, mais l'ensemble de ses interactions avec le serveur web de Buttercup Games.

L'attaquant n'a toujours pas trouvé `/passwords.pdf`. Mais son activité persistante sur 7 jours laisse supposer qu'il cherche une autre voie d'entrée.

---

## 2. Vérification des accès réussis

On commence par vérifier si cette IP a eu des requêtes abouties (status 200) :

```
index=main sourcetype="access_combined_wcookie" status=200 clientip=87.194.216.51
| table _time, status, uri
```

Nous obtenons **894 événements** avec status 200.

[![894 accès réussis — status 200](1.png)](1.png)

La conclusion change complètement par rapport à l'épisode 1. Cette IP n'est pas uniquement en reconnaissance — elle utilise activement le site. On peut voir dans les résultats :

| URI | Signification |
|-----|--------------|
| `/cart.do?action=purchase` | Achats effectués |
| `/cart/success.do` | Transactions abouties |
| `/product.screen` | Consultation de pages produits |

> ⚠️ **894 accès réussis contre seulement 20 erreurs 404.** Cette IP se fond dans le trafic légitime tout en sondant des fichiers sensibles en parallèle.

---

## 3. Analyse avec `| stats` — pages les plus accédées

Pour transformer ces 894 événements en données exploitables, on utilise `| stats` pour compter les accès par URI :

```
index=main sourcetype="access_combined_wcookie" status=200 clientip=87.194.216.51
| stats count by uri
| sort -count
| head 10
```

**Ce que fait cette commande :**

- `stats count by uri` — compte le nombre d'accès par page
- `sort -count` — trie du plus grand au plus petit (le signe `-` = décroissant)
- `head 10` — garde les 10 premiers résultats

[![Stats — top 10 des pages accédées](2.png)](2.png)

Les résultats confirment que l'IP accède principalement à des pages produits et effectue des transactions réelles.

---

## 4. Analyse avec `| top` — confirmation

On utilise `| top` pour confirmer les URIs les plus fréquentes sur l'ensemble de l'activité de cette IP, tous status confondus :

```
index=main sourcetype="access_combined_wcookie" clientip=87.194.216.51
| top uri showperc=false limit=5
```

**Ce que fait cette commande :**

- `top uri` — retourne automatiquement les valeurs les plus fréquentes avec leur count
- `showperc=false` — supprime la colonne de pourcentage
- `limit=5` — limite aux 5 premières valeurs

[![Top 5 des URIs les plus fréquentes](3.png)](3.png)

Les 5 URIs les plus fréquentes sont toutes des `/product.screen` — cette IP consulte massivement des pages produits spécifiques.

> 💡 **Différence entre `| stats` et `| top` :**
> `stats count` retourne uniquement le count, sans tri automatique.
> `top` retourne automatiquement count + percent + tri décroissant en une seule commande.

---

## 5. Analyse temporelle avec `| timechart`

Pour comprendre **quand** cette IP est active, on utilise `| timechart` pour visualiser l'évolution de son activité jour par jour sur une échelle de 7 jours :

```
index=main sourcetype="access_combined_wcookie" clientip=87.194.216.51
| timechart span=1d count by status
```

**Ce que fait cette commande :**

- `timechart` — génère une visualisation temporelle (l'axe X est **toujours** le temps)
- `span=1d` — regroupe les événements par jour
- `count by status` — crée une série par status code

[![Timechart — activité globale sur 7 jours](4.png)](4.png)



**Ce que le graphique révèle :**

| Observation | Interprétation |
|------------|---------------|
| Activité du 5 au 12 mai | 7 jours consécutifs sans interruption |
| Pic le vendredi 8 mai (~340 events) | Journée la plus active |
| Activité le week-end | Comportement atypique pour un utilisateur légitime |
| Status 200 dominant | L'activité normale masque la reconnaissance |

> ⚠️ **Une activité régulière 7j/7 sans pause week-end est un indicateur comportemental important.** Un utilisateur légitime fait généralement des pauses. Une activité continue sans interruption est atypique.

---

## 6. Vue détaillée par status code

En activant le mode **Trellis**, on obtient une vue comparative de chaque status code sur la même période :

[![Vue détaillée par status code — Trellis](6.png)](6.png)

**Ce que cette vue apporte :**

| Status | Signification | Observation |
|--------|--------------|-------------|
| 200 | Accès réussi | 74 à 344 événements/jour — activité dominante |
| 400 | Mauvaise requête | Constant, 2 à 4/jour |
| 403 | Accès interdit | Faible, 1 à 4/jour |
| 404 | Page non trouvée | Tentatives sur `/passwords.pdf` et `/hidden/` — pics irréguliers, 2 à 10/jour |
| 406 | Non acceptable | Faible, 4 à 8/jour |
| 408 | Timeout requête | Pic marqué vendredi 8 mai (~11) |
| 500 | Erreur serveur | Pic vendredi 8 mai (~8) |
| 503 | Service indisponible | Régulier, 2 à 14/jour |
| 505 | Version HTTP non supportée | Pic mercredi 7 mai (~10), puis décroissance |

> ⚠️ **Les erreurs 404 ont un pic différent des erreurs 200.** L'IP intensifie sa reconnaissance les jours où son activité "normale" est plus faible — technique délibérée pour éviter la détection.

---

## 7. Conclusion

> 🔴 **Cette IP mène une double activité simultanée.**

| Indicateur | Constat |
|-----------|---------|
| 894 accès réussis (status 200) | ✅ Utilisation active et légitime du site |
| Achats effectués (`/cart/success.do`) | ✅ Transactions réelles abouties |
| Tentatives sur `/passwords.pdf` (x3) | ❌ Ciblage répété de fichiers sensibles |
| Activité 7j/7 sans interruption | ⚠️ Comportement atypique |
| Pics de 404 décalés par rapport aux 200 | ⚠️ Reconnaissance dissimulée dans le trafic normal |

Cette IP utilise le site comme un client normal tout en sondant discrètement des fichiers sensibles en parallèle. C'est le profil d'un attaquant qui **tente de se fondre dans le trafic légitime** pour éviter la détection.

Le SOC ne peut pas déterminer avec certitude l'identité derrière cette IP. Deux hypothèses sont retenues et transmises pour escalade :

- Un **compte compromis** utilisé comme couverture par un attaquant externe
- Un **insider** qui abuse de son accès légitime pour chercher des données sensibles
---

## 8. Réponse opérationnelle SOC

### 🚧 Confinement immédiat

- Bloquer l'IP `87.194.216.51` au niveau du pare-feu
- Suspendre le compte associé aux sessions de cette IP en attendant l'investigation

### 🔔 Détection

- Créer une alerte pour toute tentative d'accès à `/passwords.pdf`
- Créer une alerte pour toute IP combinant plus de 5 erreurs 404 sur des fichiers sensibles **ET** des achats réussis
- Surveiller les IPs avec activité 7j/7 sans interruption week-end

### 🔍 Investigation complémentaire

- Identifier le compte utilisateur associé aux sessions de cette IP
- Vérifier si d'autres IPs présentent le même pattern comportemental
- Analyser les données d'achat pour détecter une éventuelle fraude

---

## 📁 Reproduire cette analyse

Le dataset utilisé est le `tutorialdata.zip` officiel de Splunk, disponible gratuitement ici :
[Télécharger tutorialdata.zip](https://docs.splunk.com/images/Tutorial/tutorialdata.zip)


---

*© Paulcyber06 — Tous droits réservés.*
