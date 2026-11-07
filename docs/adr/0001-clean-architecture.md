# ADR-0001 : Organiser la solution en Clean Architecture

- **Statut :** Accepté
- **Date :** 2026-11-06

## Contexte

CrewPlanner planifie le personnel et le matériel d'un diffuseur. Sa valeur repose
sur quelques règles métier critiques : pas de double réservation (RG-01), pas plus
de 40 h par semaine (RG-02), pas d'équipement en réparation (RG-04).

Ces règles doivent être testées souvent et rapidement, à chaque pull request.
L'application exposera une API REST et utilisera SQL Server. Elle va grossir :
4 user stories supplémentaires (US-06 à US-09) sont déjà prévues après le MVP.

Il faut donc une organisation du code qui isole les règles métier et permette de
les tester sans démarrer l'API ni la base de données.

## Décision

La solution est découpée en **4 projets .NET** selon la Clean Architecture, plus
un projet SQL séparé :

| Projet | Contenu | Dépend de |
|---|---|---|
| Domain | Entités, objets-valeurs (`PlageHoraire`), exceptions métier | Rien |
| Application | Cas d'utilisation, DTO, validation, interfaces des dépôts | Domain |
| Infrastructure | Implémentation des dépôts avec Dapper, journalisation | Application |
| Api | Contrôleurs REST, authentification, gestion des erreurs | Application, Infrastructure |
| Database (projet SQL) | Tables, procédures stockées, fonctions, déclencheurs | — |

Les dépendances pointent toujours vers Domain. Cette règle est garantie par le
compilateur : un projet ne peut utiliser que les projets qu'il référence.
Application définit les interfaces d'accès aux données, et Infrastructure les
implémente (inversion des dépendances).

## Options envisagées

**1. Architecture en 3 couches classique (Présentation → Métier → Accès aux données)**
La couche métier dépend directement de la couche d'accès aux données. Pour tester
une règle métier, il faut donc une base de données ou contourner cette dépendance.
Rejetée : c'est précisément ce que l'on veut éviter pour RG-01 à RG-04.

**2. Un seul projet API, avec la logique dans les contrôleurs ou les endpoints**
Simple et rapide à démarrer, bien adaptée à un prototype ou à un petit service.
Rejetée : les règles métier se retrouveraient réparties dans les endpoints, et
chaque test devrait démarrer l'API. Rien n'empêcherait non plus un contrôleur
d'appeler directement la base de données.

## Conséquences

**Positives**
- Domain se teste sans aucune dépendance. Application se teste en simulant les
  interfaces des dépôts, sans base de données réelle.
- Le sens des dépendances est vérifié à chaque compilation, pas seulement en revue.
- Chaque couche a un rôle clair, ce qui facilite l'ajout des futures user stories.

**Négatives**
- Plus de projets et de fichiers qu'une petite application en demanderait.
- Du code de passage entre couches : DTO, conversions, interfaces.
- Un démarrage plus lent qu'avec un seul projet.
- L'indépendance vis-à-vis de la base de données est **partielle** : une partie
  des règles est codée en T-SQL (voir ADR-0003). Changer de SGBD obligerait à
  réécrire ces procédures.