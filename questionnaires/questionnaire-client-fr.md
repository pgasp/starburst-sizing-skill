# Questionnaire de dimensionnement — Version client (FR)

*À envoyer au client avant l'appel de cadrage technique. Remplir et renvoyer par email ou partager lors de la réunion.*

---

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  DIMENSIONNEMENT PLATEFORME — QUESTIONNAIRE
  Remplissez ce que vous savez. Laissez vide ce que vous ignorez.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SECTION 1 — UTILISATEURS
─────────────────────────────────────────────────────────
1a. Combien de personnes utilisent la plateforme en même
    temps aux heures de pointe  ?
    → __________ utilisateurs simultanés

1b. Qui sont les utilisateurs principaux ? (plusieurs choix possibles)
    [ ] Utilisateurs métier / consommateurs de rapports
        (Tableau, Power BI, Looker, Qlik…)
    [ ] Analystes de données / data scientists
        (requêtes SQL exploratoires, notebooks)
    [ ] Ingénieurs data / développeurs
        (pipelines, transformations, dbt, Spark…)
    [ ] Répartition estimée : ____% métier / ____% technique


SECTION 2 — VITESSE ATTENDUE
─────────────────────────────────────────────────────────
2a. En combien de temps les requêtes doivent-elles s'exécuter ?
    [ ] Interactif — moins de 5 secondes
        (tableaux de bord en direct, rapports temps réel)
    [ ] Acceptable — 5 à 30 secondes
        (une légère attente est tolérée)
    [ ] Batch — quelques minutes acceptables
        (traitements automatisés, pas d'utilisateur en attente)

2b. Les utilisateurs lancent-ils souvent les mêmes requêtes
    ou ouvrent-ils les mêmes rapports de façon répétitive ?
    [ ] Oui — principalement des tableaux de bord fixes,
               mêmes filtres, mêmes périodes chaque jour
    [ ] Non — les requêtes varient beaucoup, imprévisibles
    [ ] Les deux


SECTION 3 — DONNÉES
─────────────────────────────────────────────────────────
3a. Volume approximatif de données à interroger :
    → __________ Go / To  (dataset total ou table la plus grande)

3b. Les utilisateurs interrogent-ils généralement
    une tranche récente des données uniquement ?
    [ ] Oui — principalement les ______ derniers mois / années
              sur ______ années d'historique total
    [ ] Non — les requêtes peuvent porter sur l'historique complet

3c. À quelle vitesse vos données augmentent-elles ?
    [ ] Stable         [ ] ~20 % par an
    [ ] >50 % par an   [ ] Inconnu


SECTION 4 — INFRASTRUCTURE
─────────────────────────────────────────────────────────
4a. Où sera hébergée la plateforme ?
    [ ] Cloud — fournisseur : [ ] AWS  [ ] Azure  [ ] GCP
    [ ] On-premise (datacenter privé / vos propres serveurs)

4b. Quelles sources de données seront connectées ?
    [ ] Lac de données / stockage objet (S3, ADLS, GCS, HDFS…)
    [ ] Bases de données relationnelles (Oracle, PostgreSQL,
        SQL Server, MySQL, Informix…)  → liste : ________________
    [ ] Les deux

4c. Combien d'environnements sont nécessaires ?
    [ ] Production uniquement
    [ ] Production + Recette / Test
    [ ] Production + Recette + Développement
    [ ] Autre : __________


SECTION 5 — DISPONIBILITÉ & CONTINUITÉ
─────────────────────────────────────────────────────────
5a. En cas d'incident, combien de temps la plateforme
    peut-elle être indisponible sans impact grave ?
    [ ] Pas du tout — doit rester opérationnelle en permanence
        (trading, opérations critiques, obligation réglementaire)
    [ ] Jusqu'à 15–30 minutes — bascule automatique acceptable
    [ ] Quelques heures — intervention manuelle acceptable
    [ ] Pas d'exigence particulière

5b. Des obligations réglementaires ou contractuelles
    s'appliquent-elles à la disponibilité ?
    [ ] Oui — préciser : ____________________
    [ ] Non / Inconnu

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
