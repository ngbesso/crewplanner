# ADR-0002 : Utiliser Dapper plutôt qu'EF Core pour l'accès aux données

- **Statut :** Accepté
- **Date :** 2026-11-07

## Contexte

Dans le cadre du projet CrewPlanner, les règles de gestion métier (RG-01 à RG-05) sont
appliquées dans la base de données, par des procédures stockées, une fonction, un
déclencheur et des contraintes en T-SQL. De plus, le schéma de la base de données et son
cycle de vie sont intégralement gérés par un projet SQL Server Data Tools (SSDT).

L'accès aux données consiste donc principalement à exécuter ces procédures stockées et à
convertir leurs résultats en objets C#. Enfin, la maîtrise du langage T-SQL constitue un
objectif technique explicite pour l'équipe sur ce projet.

## Décision

Nous choisissons d'utiliser Dapper comme outil unique d'accès aux données pour CrewPlanner.

Dapper sera exclusivement encapsulé dans la couche Infrastructure, où il implémentera les
interfaces de dépôts (repositories) définies dans la couche Application, en cohérence
avec l'ADR-0001.

## Options envisagées

**1. Entity Framework Core (EF Core)**

Un ORM complet dont les forces principales sont la génération automatique de requêtes via
LINQ, le suivi des modifications des entités (change tracking) et la gestion des versions
du schéma via les migrations.

- Raison du rejet : les atouts majeurs d'EF Core ne correspondent pas aux choix techniques
  de CrewPlanner. Le schéma étant piloté par SSDT, les migrations d'EF Core sont inutiles.
  Le suivi des modifications apporte peu de valeur puisque les écritures passent par des
  procédures stockées. Utiliser EF Core ici ajouterait une couche de complexité sans
  exploiter ses fonctionnalités phares.

**2. EF Core et Dapper ensemble**

Une combinaison courante en entreprise : EF Core pour les opérations CRUD simples, Dapper
pour les procédures stockées et les requêtes complexes.

- Raison du rejet : deux outils d'accès aux données à configurer, à apprendre et à
  maintenir, pour un projet de cette taille. Comme presque toutes les écritures passent
  par des procédures stockées, la part de CRUD simple confiée à EF Core serait trop
  faible pour justifier cette complexité.

## Conséquences

**Positives**

- Cohérence technologique : alignement avec le projet SQL (SSDT) et l'usage intensif
  des procédures stockées.
- Simplicité de la couche Infrastructure : le code se limite à l'ouverture de la
  connexion, la transmission des paramètres et l'appel de la méthode d'extension de Dapper.
- Développement des compétences : renforcement de l'expertise de l'équipe en T-SQL.

**Négatives**

- Absence de validation à la compilation : le texte des requêtes SQL et les noms des
  procédures stockées ou de leurs paramètres sont manipulés sous forme de chaînes de
  caractères. Les fautes de frappe ou les désynchronisations avec la base de données ne
  provoquent pas d'erreur de compilation. Les résultats, eux, restent fortement typés.
  *Atténuation :* les tests d'intégration (Testcontainers) exécutent chaque appel de
  procédure contre un vrai SQL Server dans la CI, ce qui détecte ces erreurs avant la
  fusion.
- Code répétitif : il faut écrire manuellement le code de chaque appel de procédure,
  notamment pour associer les paramètres d'entrée et de sortie.