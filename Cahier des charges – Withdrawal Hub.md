# Cahier des charges – Withdrawal Hub

## 1. Contexte

Withdrawal Hub est le système d'information du collectif AI4ALL, qui organise un festival consacré à l'intelligence artificielle. Le projet se limite à un besoin précis : remplacer vite un exposant ou un animateur qui se désiste.

Le collectif compte une vingtaine de membres (à confirmer) : organisateurs, exposants et animateurs. Chaque intervention occupe une journée entière, de 10h00 à 18h30, sur un stand donné. Aujourd'hui, un désistement passe par des messages dispersés et le planning est refait à la main.

## 2. Besoin, problématique et objectif

**Problématique :** comment permettre au collectif AI4ALL de retrouver rapidement un remplaçant lorsqu'un exposant ou un animateur se désiste, sans perdre de temps en échanges manuels ?

**Objectif :** réduire le temps entre un désistement et son remplacement confirmé, en automatisant le parcours suivant :

1. la personne se désiste en indiquant un motif ;
2. les organisateurs sont prévenus par e-mail ;
3. les personnes disponibles ce jour-là sont identifiées et alertées ;
4. un remplaçant est affecté en un clic (premier arrivé, premier servi) ;
5. les confirmations partent par e-mail et le planning est mis à jour.

**Hors périmètre du MVP :** billetterie, gestion des visiteurs, facturation, planning complet du festival, application mobile.

## 3. Utilisateurs et droits d'accès

Quatre profils utilisent l'application ; les visiteurs du site vitrine n'ont aucun compte.

| Profil | Besoins principaux | Droits dans l'application |
| --- | --- | --- |
| Organisateur | Traiter les désistements, alerter, affecter, suivre les indicateurs | Tout voir et tout traiter : file à pourvoir, candidats, alertes, affectations, KPI |
| Animateur | Consulter son planning, se désister, indiquer ses disponibilités, remplacer | Ses interventions et celles de son stand ; ses disponibilités |
| Exposant | Voir l'équipe de son stand et signaler un désistement | Les interventions de son stand |
| Administrateur SI | Superviser les services, gérer les comptes, traiter les tickets | Accès aux serveurs et à l'outil de tickets, pas aux données métier par défaut |
| Visiteur | Découvrir AI4ALL, contacter le collectif | Site vitrine public uniquement |

Les droits sont appliqués par le serveur (jeton JWT et profil) : un animateur qui appelle une route réservée aux organisateurs est refusé.

## 4. Fonctionnalités

**Comptes et authentification**

- connexion par e-mail et mot de passe (haché) ;
- trois rôles métier : organisateur, exposant, animateur ;
- accès limité selon le rôle.

**Planning**

- consulter son planning mensuel (statut : confirmé, à remplacer, remplacé) ;
- ouvrir la fiche d'une journée : stand, équipe, matériel, consignes.

**Désistement**

- se désister d'une intervention à venir avec un motif (malade, imprévu personnel, indisponibilité professionnelle, autre) et des précisions ;
- un seul désistement actif par intervention ;
- retirer son désistement tant qu'aucun remplaçant n'est affecté ;
- e-mail automatique aux organisateurs.

**Remplacement (organisateur)**

- file « à pourvoir » avec le nombre de candidats disponibles ;
- liste des candidats, le meilleur choix en premier (habitué du stand, peu chargé) ;
- bouton « alerter tous les disponibles » : un e-mail avec lien personnel par candidat ;
- affectation directe en un clic ;
- acceptation depuis l'e-mail, premier arrivé premier servi ;
- e-mails de confirmation au remplaçant et à la personne désistée.

**Disponibilités**

- chaque animateur déclare les jours où il peut remplacer.

**Indicateurs**

- désistements du mois, taux de remplacement, délai moyen d'alerte et de résolution, répartition par motif.

## 5. Contraintes et architecture

L'infrastructure tient sur quatre machines virtuelles, dimensionnées pour 30 utilisateurs et extensibles. Les choix ci-dessous sont des propositions à justifier dans le dossier final.

| Machine | Rôle | Services |
| --- | --- | --- |
| srv-web | Point d'entrée Internet | Nginx (HTTPS), site vitrine, front de l'application |
| srv-app | Application | API Node.js / Express en conteneur Docker |
| srv-db | Données | PostgreSQL 16, sauvegardes chiffrées |
| srv-outils | Exploitation | Outil de tickets ITIL, supervision, analyse de trafic |

**Service externe :** l'API transactionnelle Brevo envoie les e-mails (alerte, confirmation). Un e-mail en échec n'annule jamais l'action de l'utilisateur.

**Site vitrine :** accessible depuis Internet, responsive, il présente le collectif, le festival et un formulaire de contact.

**Sécurité :**

- HTTPS partout, pare-feu : seul `srv-web` est exposé, les autres machines ne sont joignables que depuis le réseau interne ;
- mots de passe hachés, jetons JWT, jetons à usage unique dans les e-mails d'alerte ;
- accès administrateur limité (clé SSH, pas de mot de passe) ;
- règles d'utilisation et de sécurité écrites (mots de passe, e-mails, données personnelles) et remises à chaque membre.

**Données :** les données personnelles (nom, e-mail, disponibilités) sont limitées au nécessaire et accessibles selon le profil.

**Sauvegarde :** export PostgreSQL quotidien, copie sur un second emplacement, conservation de plusieurs jours. Une procédure de restauration est écrite et testée ; le résultat du test est documenté.

**Continuité :** les services critiques sont l'API et la base. Ils redémarrent automatiquement, et la restauration depuis la sauvegarde sert de plan de reprise. L'objectif est de rétablir le service en moins d'une heure (à valider).

**Supervision :** surveillance de la disponibilité des services (sonde `/health` de l'API, site vitrine) et des ressources des machines, avec alerte en cas de panne.

**Évolutivité et documentation :** conteneurs et schéma de base versionnés ; l'architecture, le réseau, les droits, la sauvegarde et les procédures de reprise sont documentés.

## 6. Exploitation : tickets, incidents et support

Un outil de tickets open source (GLPI proposé, à valider) centralise toutes les demandes selon les principes ITIL : chaque ticket est créé, classé, affecté, traité, puis clôturé avec une trace.

| Type de ticket | Exemple | Qui traite |
| --- | --- | --- |
| Demande utilisateur | Créer un compte, changer un rôle, corriger une disponibilité | Administrateur SI |
| Incident système et réseau | Serveur injoignable, certificat expiré, sauvegarde en échec | Administrateur SI |
| Incident applicatif | E-mail d'alerte non reçu, affectation refusée à tort | Administrateur SI, puis développeur |

- Priorité = impact × urgence. Un désistement non remplaçable la veille d'un événement est traité en premier.
- Statuts : nouveau, affecté, en cours, résolu, clôturé. Chaque incident résolu reçoit une cause et une solution documentées.
- **Accompagnement :** guide de prise en main illustré (captures de l'application), résumé d'une page par profil, courte démonstration aux membres avant la mise en service, et contact de support indiqué dans l'application.

## 7. Tests, mise en production et indicateurs

**Tests et validation.** Chaque service est testé avant la mise en production et les résultats sont consignés dans un tableau (cas, résultat attendu, résultat obtenu, date).

| Test | Attendu |
| --- | --- |
| Connexion avec un mauvais mot de passe | Refus, message clair |
| Animateur sur une route réservée aux organisateurs | Accès refusé |
| Désistement, puis second désistement sur la même intervention | Le second est refusé |
| Alerte de tous les disponibles | Un e-mail par candidat |
| Deux acceptations simultanées | La première gagne, la seconde reçoit « déjà pourvue » |
| Affectation d'une personne non disponible | Refusée |
| Restauration d'une sauvegarde | Données retrouvées à l'identique |

Les six premiers cas ont déjà été joués avec succès sur l'API de développement ; la restauration reste à faire.

**Mise en production.** Préparation (liste de contrôle : secrets, HTTPS, sauvegarde active, supervision), déploiement par Docker Compose, vérification après déploiement, puis compte rendu documenté avec possibilité de retour arrière.

**Indicateurs de résultat (dans l'application) :** nombre de désistements, taux de remplacement, délai moyen d'alerte, délai moyen de résolution, répartition par motif.

**Analyse du trafic :** nombre de visiteurs du site vitrine, pages vues, connexions à l'application, disponibilité mesurée des services.

**Suivi du projet :** tâches terminées sur tâches prévues, jalons tenus, tickets ouverts et résolus.

## 8. Planning et livrables

Le planning est indicatif : la mise en production est visée le 25 novembre 2026. Le suivi se fait dans un outil de gestion de projet (tableau de tâches avec jalons).

&#91;embedded content: planning · 8 lots, 1 jalon\]

**Livrables :** maquette Figma, modèle de données, API, application web, site vitrine, documentation technique, tableau de tests, rapport de restauration, guides utilisateurs.

## 9. Correspondance avec les 18 contraintes du sujet

| N° | Contrainte | Réponse du projet | Section |
| --- | --- | --- | --- |
| 1 | Infrastructure | 4 machines virtuelles, réseau interne documenté | 5 |
| 2 | Recensement des ressources | Inventaire des machines, services, comptes et licences | 5 |
| 3 | Site vitrine | Site public responsive avec formulaire de contact | 5 |
| 4 | Application web | Withdrawal Hub : API, base PostgreSQL, service externe Brevo | 4, 5 |
| 5 | Gestion des tickets | Outil de tickets selon ITIL | 6 |
| 6 | Gestion des accès | Profils et droits appliqués par le serveur | 3 |
| 7 | Continuité de service | Services critiques identifiés, redémarrage, plan de reprise | 5 |
| 8 | Sauvegarde | Export quotidien, copie externe, restauration testée | 5, 7 |
| 9 | Règles d'utilisation et de sécurité | Charte remise à chaque membre | 5 |
| 10 | Demandes utilisateurs | Tickets de type demande | 6 |
| 11 | Incidents système et réseau | Tickets d'incident, cause et solution documentées | 6 |
| 12 | Incidents applicatifs | Tickets d'incident applicatif | 6 |
| 13 | Analyse du trafic | Statistiques du site, connexions, disponibilité | 7 |
| 14 | Planification du projet | Outil de gestion de projet, planning | 8 |
| 15 | Indicateurs de suivi | KPI de résultat et d'avancement | 7 |
| 16 | Tests et validation | Tableau de tests documenté | 7 |
| 17 | Mise en production | Liste de contrôle, déploiement, compte rendu | 7 |
| 18 | Accompagnement des utilisateurs | Guide, résumés par profil, démonstration | 6 |
