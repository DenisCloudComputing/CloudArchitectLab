# CloudArchitectLab

## Objectif du Projet

Ce projet constitue un laboratoire pratique visant à maîtriser le cycle de vie DevOps complet, depuis le versionnement de code sur un poste local jusqu'à son déploiement automatisé sur une infrastructure cloud AWS. L'objectif est de démontrer une compréhension opérationnelle des outils Git, GitHub, SSH et AWS EC2, en se concentrant sur la sécurité des connexions et la documentation rigoureuse des procédures de dépannage.

Ce dépôt sert de portfolio technique pour démontrer les compétences suivantes :
- Gestion de versions avec Git et GitHub.
- Déploiement d'infrastructure cloud avec AWS EC2.
- Sécurisation de tous les flux réseau par authentification par clés SSH.
- Résolution de problèmes complexes liés aux permissions et à la mise en réseau.
- Documentation technique de niveau production.

## Architecture Technique

Le flux de déploiement est structuré en trois environnements distincts :
1.  **Poste de développement local (Windows 11 / Git Bash)** : Développement du code et gestion de version.
2.  **Dépôt central (GitHub)** : Hébergement du code source et de son historique.
3.  **Serveur cible (AWS EC2 Ubuntu 26.04 LTS)** : Exécution du code déployé.

### Gestion des Identités et des Accès

La sécurité du flux repose sur trois paires de clés SSH distinctes, appliquant le principe du moindre privilège à chaque segment du réseau.

- **Connexion 1 (Poste Local vers GitHub)**
  - **Clé locale** : `~/.ssh/id_ed25519`
  - **Clé distante** : Ajoutée dans les paramètres GitHub.
  - **Rôle** : Authentifier le poste de développement pour les opérations `git push` et `git pull`.

- **Connexion 2 (Poste Local vers EC2)**
  - **Clé locale** : `CloudArchitectLab-Key.pem` (générée et téléchargée depuis AWS).
  - **Clé distante** : Conservée par AWS et associée à l'instance EC2.
  - **Rôle** : Authentifier l'administrateur pour l'accès SSH au serveur.

- **Connexion 3 (Instance EC2 vers GitHub)**
  - **Clé locale** : `~/.ssh/id_ed25519` (générée directement sur l'instance EC2).
  - **Clé distante** : Ajoutée dans les paramètres GitHub.
  - **Rôle** : Authentifier le serveur pour l'opération de `git clone` du dépôt.

## Guide de Déploiement

### 1. Configuration de l'Environnement Local

Création du dépôt Git et première version du script de configuration serveur.

```bash
mkdir CloudArchitectLab && cd CloudArchitectLab
git init
touch setup-webserver.sh
git add setup-webserver.sh
git commit -m "feat: initial commit with empty web server setup script"
2. Liaison avec le Dépôt Distant GitHub
Génération de la clé d'identité locale et configuration du dépôt distant.

bash
ssh-keygen -t ed25519 -C "votre-email@example.com"
# Contenu de la clé publique à copier dans GitHub > Settings > SSH keys
cat ~/.ssh/id_ed25519.pub

ssh -T git@github.com
git remote add origin git@github.com:DenisCloudComputing/CloudArchitectLab.git
git push -u origin main
3. Provisionnement de l'Infrastructure Cloud
Lancement d'une instance EC2 avec les spécifications suivantes :

AMI : Ubuntu Server 26.04 LTS (éligible à l'offre gratuite).

Type : t2.micro.

Paire de clés : CloudArchitectLab-Key (fichier .pem sauvegardé localement).

Groupe de sécurité : Règle entrante pour SSH (port 22) autorisée uniquement depuis "Mon adresse IP".

4. Connexion au Serveur et Déploiement
Une fois l'instance en cours d'exécution, connexion et exécution du script.

bash
# Depuis le répertoire local contenant la clé .pem
ssh -i "CloudArchitectLab-Key.pem" ubuntu@<DNS_PUBLIQUE_INSTANCE>

# Opérations sur le serveur
ssh-keygen -t ed25519 -C "ec2-instance-cloudarchitectlab"
cat ~/.ssh/id_ed25519.pub
# Ajouter la clé publique affichée sur GitHub (Settings > SSH keys)

mkdir ~/projects && cd ~/projects
git clone git@github.com:DenisCloudComputing/CloudArchitectLab.git
cd CloudArchitectLab
chmod +x setup-webserver.sh
sudo ./setup-webserver.sh
Dépannage : Diagnostic et Résolution de Problèmes
Cette section documente les obstacles techniques rencontrés lors de la phase de connexion, l'analyse de leurs causes profondes et les solutions appliquées.

1. Échec de Connexion SSH lors de l'Utilisation de l'Adresse IP Publique
Symptôme :
La commande ssh -i "cle.pem" ubuntu@<adresse_ip_publique> retourne une erreur de connexion (Connection refused ou Permission denied (publickey)).

Diagnostic :
L'adresse IPv4 publique d'une instance EC2 est dynamique par défaut. Elle est susceptible de changer lors de l'arrêt puis du redémarrage de l'instance. Par ailleurs, il a été observé que certaines configurations réseau internes à AWS peuvent rendre la connexion directe par IP moins fiable que l'utilisation du nom DNS public fourni par AWS. L'utilisation de l'IP ne permet pas non plus de bénéficier pleinement du mécanisme de vérification d'empreinte du serveur (host key fingerprint).

Solution Appliquée :
Utiliser le nom DNS public de l'instance, fourni dans la console AWS EC2, pour toutes les connexions SSH. Ce nom est plus stable et est automatiquement résolu par l'infrastructure AWS vers l'IP correcte.

bash
# Méthode fiable : Utilisation du DNS public AWS
ssh -i "CloudArchitectLab-Key.pem" ubuntu@ec2-34-235-162-29.compute-1.amazonaws.com
2. Gestion des Permissions de la Clé Privée sous Windows
Symptôme :
La tentative de restriction des permissions de la clé privée avec chmod 400 dans Git Bash échoue. Le fichier conserve des permissions trop ouvertes (par exemple, -r--r--r--), ce qui est refusé par le client SSH.

Diagnostic :
Git Bash émule un environnement Unix sur le système de fichiers Windows NTFS. La commande chmod est conçue pour les systèmes de fichiers Unix (ext4, XFS, etc.) qui gèrent nativement les permissions en mode rwx pour le propriétaire, le groupe et les autres. NTFS utilise un système de listes de contrôle d'accès (ACLs) fondamentalement différent. La traduction effectuée par Git Bash entre ces deux systèmes est imparfaite et ne permet pas de définir un mode 400 strict sur un fichier situé dans le répertoire utilisateur Windows.

Solution Appliquée :
Utiliser l'outil natif de gestion des ACLs de Windows, icacls.exe, depuis une console PowerShell. L'opération a été décomposée en deux étapes pour garantir la fiabilité sur différentes versions de Windows.

Suppression de l'héritage : Isoler le fichier des permissions de son dossier parent.

Attribution de droits stricts : Révoquer toutes les permissions et n'accorder que le droit de lecture au propriétaire courant.

powershell
# Étape 1 : Supprimer toutes les permissions héritées du dossier parent
icacls C:\Users\Denis\aws-keys\CloudArchitectLab-Key.pem /inheritance:r

# Étape 2 : Accorder le droit de lecture uniquement à l'utilisateur "Denis"
icacls C:\Users\Denis\aws-keys\CloudArchitectLab-Key.pem /grant:r "Denis:(R)"
3. Résolution du Nom de Fichier et du Chemin d'Accès
Symptôme :
Des difficultés ont été rencontrées pour faire référence au fichier de clé .pem en utilisant un chemin absolu (ex: /c/Users/Denis/aws-keys/...).

Diagnostic :
La combinaison de l'interprétation des chemins par Git Bash et de la présence potentielle de caractères spéciaux ou d'espaces peut entraîner une résolution incorrecte du chemin d'accès au fichier. Le fait que la commande soit exécutée depuis le répertoire du projet (~/CloudArchitectLab) ajoute à la confusion, car la clé n'y est pas présente.

Solution Appliquée :
Simplifier la commande en changeant le répertoire de travail courant pour se placer directement dans le dossier contenant la clé. Cela permet d'utiliser un chemin relatif (le simple nom du fichier), plus fiable et moins sujet aux erreurs d'interprétation.

bash
# Changement du répertoire de travail avant la connexion
cd ~/aws-keys/
ssh -i "CloudArchitectLab-Key.pem" ubuntu@<DNS_PUBLIQUE_INSTANCE>
Récapitulatif des Commandes Git Essentielles
git init : Initialise un nouveau dépôt Git local.

git status : Affiche l'état des fichiers (modifiés, indexés, non suivis).

git add <fichier> : Ajoute un fichier à la zone d'index (staging area).

git commit -m "<message>" : Enregistre les modifications indexées dans l'historique.

git diff : Montre les différences entre le répertoire de travail et l'index.

git diff --staged : Montre les différences entre l'index et le dernier commit.

git log --oneline : Affiche l'historique des commits de manière condensée.

git checkout -b <branche> : Crée une nouvelle branche et bascule dessus.

git branch -D <branche> : Supprime une branche localement.

git remote add origin <url> : Associe le dépôt local à un dépôt distant.

git push -u origin main : Pousse la branche locale 'main' vers le dépôt distant 'origin' et définit le suivi.

git clone <url> : Copie un dépôt distant sur la machine locale.

Prochaines Étapes (Lab Partie 2)
La suite de ce laboratoire se concentrera sur l'intégration et le déploiement continus (CI/CD) en intégrant les composants suivants :

Transformation du script shell en un playbook Ansible pour une configuration plus robuste.

Mise en place d'un pipeline CI/CD avec GitHub Actions pour tester et déployer automatiquement les modifications.

Ajout d'un fichier index.html pour servir une page web personnalisée.

Configuration d'un enregistrement DNS et d'un certificat HTTPS avec Let's Encrypt.
