# DevOps-AppGestionDesProjets

Ce dépôt contient un projet **Spring Boot** (backend) et **Angular** (frontend) utilisé dans le cadre du module **DevOps**.  
Il sert de support pratique pour mettre en œuvre un pipeline Jenkins basé sur l'intégration continue (CI), la livraison continue (Continuous Delivery) et le déploiement continu (Continuous Deployment).

---

## 📁 Structure du projet

```
DevOps-AppGestionDesProjets/
├── backend/          # API REST Spring Boot (Java 17/21, Maven)
│   ├── src/
│   │   └── main/java/tn/esprit/backend/
│   │       ├── entity/
│   │       ├── repository/
│   │       ├── service/
│   │       └── controller/
│   ├── mvnw          # Maven Wrapper exécutable
│   └── pom.xml
├── frontend/         # Application Angular 22 (Node.js, npm)
│   ├── src/
│   └── package.json
├── Jenkinsfile       # Pipeline Déclaratif Jenkins CI/CD complet
└── README.md
```

---

## 🧩 Modèle de données

Le projet simule une application de gestion de projets pour une organisation composée d'entreprises, d'équipes et de projets.

| Entité | Description |
|---|---|
| `Entreprise` | Possède un nom et une adresse, regroupe plusieurs équipes |
| `Equipe` | Appartient à une entreprise, a un nom et une spécialité, travaille sur plusieurs projets |
| `Projet` | Sujet de travail partagé entre plusieurs équipes |
| `ProjetDetaille` | Détails techniques d'un projet : technologie, coût provisoire, date de début |

---

## 🔌 Endpoints REST du Backend

| Contrôleur | Méthode | URL | Description |
|---|---|---|---|
| **Entreprise** | GET | `/entreprise/all` | Lister toutes les entreprises |
| | GET | `/entreprise/get/{id}` | Obtenir une entreprise par ID |
| | POST | `/entreprise/add` | Ajouter une entreprise |
| | PUT | `/entreprise/update` | Modifier une entreprise |
| | DELETE | `/entreprise/delete/{id}` | Supprimer une entreprise |
| **Equipe** | GET | `/equipe/all` | Lister toutes les équipes |
| | GET | `/equipe/get/{id}` | Obtenir une équipe par ID |
| | POST | `/equipe/add` | Ajouter une équipe |
| | PUT | `/equipe/update` | Modifier une équipe |
| | DELETE | `/equipe/delete/{id}` | Supprimer une équipe |
| | PUT | `/equipe/assign-entreprise/{equipeId}/{entrepriseId}` | Affecter une équipe à une entreprise |
| | PUT | `/equipe/assign-projet/{equipeId}/{projetId}` | Affecter une équipe à un projet |
| **Projet** | GET | `/projet/all` | Lister tous les projets |
| | GET | `/projet/get/{id}` | Obtenir un projet par ID |
| | POST | `/projet/add` | Ajouter un projet |
| | PUT | `/projet/update` | Modifier un projet |
| | DELETE | `/projet/delete/{id}` | Supprimer un projet |
| **Projet Détaillé** | GET | `/projet-detaille/all` | Lister tous les projets détaillés |
| | GET | `/projet-detaille/get/{id}` | Obtenir un projet détaillé par ID |
| | POST | `/projet-detaille/add` | Ajouter un projet détaillé |
| | PUT | `/projet-detaille/update` | Modifier un projet détaillé |
| | DELETE | `/projet-detaille/delete/{id}` | Supprimer un projet détaillé |

---

## 🚀 Pipeline CI/CD Jenkins & Déploiement Continu

Le projet implémente une chaîne CI/CD complète, sécurisée et automatisée dans `Jenkinsfile`.

### 1. Cycle de vie des étapes Jenkins (Stage View)

```
[ Checkout ] ➔ [ Clean ] ➔ [ Test ] ➔ [ Package ] ➔ [ Archive Artifacts ] ➔ [ Deploy ]
```

1. **`Checkout`** : Récupère le code source de la branche `ci/jenkins-backend` via Git SCM.
2. **`Clean`** : Nettoie l'espace de compilation (`./mvnw clean`) pour garantir l'idempotence du build.
3. **`Test`** : Exécute les tests unitaires Spring Boot (`./mvnw test`) avec injection sécurisée des identifiants MySQL (`mysql-backend-db`) et publie les rapports JUnit (`junit 'backend/target/surefire-reports/*.xml'`).
4. **`Package`** : Empaquète l'application en archive JAR exécutable (`./mvnw -DskipTests package`) sans réexécuter les tests une seconde fois.
5. **`Archive Artifacts` (Continuous Delivery)** : Archive le binaire `backend/target/backend-1.0.0.jar` avec calcul de l'empreinte cryptographique MD5 (`fingerprint: true`). L'artefact est téléchargeable directement sur la page du build Jenkins.
6. **`Deploy` (Continuous Deployment)** : Déploie automatiquement le binaire JAR en local dans WSL sur le **port 8082** avec remplacement propre de l'ancienne instance et vérification de disponibilité HTTP.

### 2. Déclencheurs Automatiques (Triggers)

Le pipeline configure deux mécanismes de déclenchement complémentaires :

* **Webhook GitHub (Événementiel)** : `githubPush()`  
  Chaque `git push` sur la branche `ci/jenkins-backend` déclenche immédiatement et automatiquement le pipeline via le Webhook GitHub connecté au tunnel ngrok (`/github-webhook/`).
* **Cron Planifié (Temporel)** : `cron('H 2 * * 1-5')`  
  Exécute un build automatique de non-régression chaque nuit du lundi au vendredi entre 02h00 et 02h59 (heure locale WAT / UTC+1).

### 3. Architecture des Ports et Sécurité du Déploiement

* **Port 8080** : Réservé exclusivement à **Jenkins**.
* **Port 8082** : Dédié au **Backend Spring Boot déployé**.  
  *Justification :* Évite tout conflit de socket réseau avec Jenkins sur la même machine WSL.
* **Port 3306** : Serveur de base de données **MySQL** (`test_db`).
* **Secrets** : Aucun mot de passe ni identifiant n'est écrit dans le code, les logs ou l'historique shell. Les variables de connexion (`SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`) sont injectées au runtime via le gestionnaire de credentials Jenkins `mysql-backend-db`.

### 4. Vérification du Backend Déployé

Une fois le pipeline exécuté avec succès, vérifier l'état du backend sur le port **8082** :

```bash
# Tester l'API (doit retourner HTTP 200 avec un tableau JSON [])
curl -i http://localhost:8082/equipe/all
curl -i http://localhost:8082/entreprise/all
curl -i http://localhost:8082/projet/all

# Vérifier le port d'écoute 8082
ss -tlnp | grep 8082

# Consulter les logs d'exécution du backend
tail -f /var/lib/jenkins/backend-deploy/backend-app.log
```

### 5. Gestion du Processus Déployé (Arrêt / Redémarrage Manuel)

Le processus backend est géré via son fichier PID stocké dans `/var/lib/jenkins/backend-deploy/backend.pid` :

```bash
# Arrêter proprement le backend déployé
kill $(cat /var/lib/jenkins/backend-deploy/backend.pid)

# Redémarrer manuellement le backend déployé
export SERVER_PORT=8082
nohup java -Dserver.port=8082 -jar /var/lib/jenkins/backend-deploy/backend-1.0.0.jar > /var/lib/jenkins/backend-deploy/backend-app.log 2>&1 &
echo $! > /var/lib/jenkins/backend-deploy/backend.pid
```

---

## 🛠️ Stack technique

| Couche | Technologie | Port / Service |
|---|---|---|
| Backend | Spring Boot 4.1.0, Spring Data JPA, Hibernate | Port `8082` |
| Base de données | MySQL 8.0 Server | Port `3306` (`test_db`) |
| CI/CD Server | Jenkins 2.580.2 | Port `8080` |
| Tunnel Webhook | ngrok v3 | HTTPS ➔ Localhost:8080 |
| Java Runtime | OpenJDK 21 | Tool `JDK21` |
| Build Tool | Maven Wrapper (`./mvnw`) | Maven 3.9+ |
| SCM | Git / GitHub | Branche `ci/jenkins-backend` |

---

## 👤 Auteur

**Ahmed Mhirsi** — ESPRIT  
DevOps Academic Project
