# TP : Déploiement Continu sur AWS & Automatisation avec Terraform & Ansible

## Contexte

Ce TP est la **suite directe** du TP CI/CD avec GitHub Actions. Dans le TP précédent, le déploiement était **simulé** (des `echo` dans le terminal). Cette fois, vous allez déployer **réellement** votre API Node.js sur un vrai serveur dans le cloud, puis automatiser la configuration de ce serveur avec Ansible.

> ⚠️ **Prérequis :**
> - Avoir terminé le TP CI/CD GitHub Actions (pipeline CI fonctionnel)
> - Avoir un compte **AWS Academy** avec accès au **Cloud foundation** via Vocareum
> - Votre dépôt `tp-api-nodejs` sur GitHub avec les workflows CI en place

## Durée : 3 heures

---

# 🤔 PARTIE 0 : Pourquoi tout ça ?

## Le problème

Dans le TP précédent, votre workflow `deploy.yml` faisait ceci :

```bash
echo "🚀 Déploiement staging terminé !"
echo "🌐 URL : https://tp-api-nodejs-staging.example.com"
```

C'était utile pour apprendre, mais **rien n'était réellement déployé**. Personne ne pouvait ouvrir un navigateur et accéder à votre API.

## Ce qu'on va faire dans ce TP

On va mettre votre API **en ligne pour de vrai**, sur un serveur dans le cloud. Voici le plan en 3 grandes étapes :

```
1. TERRAFORM  → Créer le serveur (comme commander un ordinateur en ligne)
2. ANSIBLE    → Installer les logiciels sur le serveur (Node.js, PM2...)
3. GITHUB     → Déployer automatiquement à chaque push
   ACTIONS
```

## Les 3 outils et leur rôle

Pensez à la construction d'une maison :

| Outil | Rôle | Analogie maison |
|-------|------|-----------------|
| **Terraform** | Créer le serveur, ouvrir les ports réseau | Construire les murs, installer les portes et fenêtres |
| **Ansible** | Installer Node.js, PM2, configurer le serveur | Meubler la maison, installer l'électricité |
| **GitHub Actions** | Déployer le code automatiquement | Emménager et ranger les affaires |

## AWS Academy et Vocareum

Votre compte AWS Academy vous donne accès à un **Learner Lab** via la plateforme Vocareum.

Ce qu'il faut retenir :

- Vous avez **~100$ de crédits** (largement suffisant)
- Vos **identifiants AWS expirent** à chaque fin de session — il faudra les renouveler
- La **région** à utiliser est `us-east-1` (N. Virginia)

> ⚠️ **Si quelque chose ne marche plus**, c'est souvent parce que vos identifiants Vocareum ont expiré. Relancez le lab (Stop Lab → Start Lab).

**❓ Questions de compréhension :**
1. Quelle est la différence entre un déploiement simulé et un déploiement réel ?
2. Pourquoi les identifiants temporaires sont-ils plus sûrs que des identifiants permanents ?

---

# 🏗️ PARTIE 1 : Créer le Serveur avec Terraform

## Pourquoi Terraform plutôt que cliquer dans AWS ?

Vous pourriez créer votre serveur en cliquant dans la console AWS (l'interface web). Mais imaginez :

- Vous faites une erreur → il faut tout recommencer en cliquant
- Vous voulez la même chose pour un collègue → vous devez lui écrire un tutoriel avec des captures d'écran
- Vous voulez tout supprimer → il faut retrouver chaque ressource et la supprimer une par une

**Terraform** résout ces problèmes : vous écrivez ce que vous voulez dans des fichiers texte, et Terraform s'occupe de tout créer (ou tout supprimer) pour vous.

```
Sans Terraform                    Avec Terraform
──────────────                    ──────────────
😰 20 clics dans AWS              😊 terraform apply
📸 Tutoriel = captures d'écran    📄 Tutoriel = le code lui-même
🗑️ Supprimer = retrouver chaque   🗑️ terraform destroy
   ressource une par une              → tout supprimé d'un coup
```

## Étape 1.1 : Démarrer votre lab AWS

1. Connectez-vous à **AWS Academy** via Vocareum
2. Cliquez sur **Start Lab** et attendez que le voyant passe au **vert 🟢**
3. Cliquez sur **AWS Details** — vous y trouverez vos identifiants AWS (on en aura besoin bientôt)

> 💡 **Le voyant vert** = votre lab est actif. **Voyant rouge** = vos identifiants ont expiré.

## Étape 1.2 : Installer Terraform sur votre ordinateur

Terraform est un programme que vous installez **sur votre machine** (pas sur le serveur AWS).

**Choisissez votre système d'exploitation :**

<details>
<summary>🍎 macOS</summary>

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```
</details>

<details>
<summary>🐧 Linux (Ubuntu/Debian)</summary>

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```
</details>

<details>
<summary>🪟 Windows</summary>

```powershell
choco install terraform
```
Ou téléchargez depuis https://developer.hashicorp.com/terraform/downloads
</details>

**Vérifiez que ça a marché :**

```bash
terraform --version
# Vous devriez voir : Terraform v1.x.x
```

✅ Si vous voyez un numéro de version, c'est bon !

## Étape 1.3 : Créer une clé SSH

Pour se connecter à votre futur serveur, il faut une **clé SSH**. C'est comme une clé physique : vous gardez la clé privée (secrète), et vous mettez la clé publique sur la serrure du serveur.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/tp-api-keypair -N ""
```

> 💡 **Que fait cette commande ?**
> - `ssh-keygen` = outil pour générer des clés SSH
> - `-t rsa -b 4096` = type de clé (RSA) avec une sécurité forte (4096 bits)
> - `-f ~/.ssh/tp-api-keypair` = nom et emplacement du fichier
> - `-N ""` = pas de mot de passe supplémentaire (pour simplifier le TP)

Deux fichiers sont créés :
- `~/.ssh/tp-api-keypair` → la **clé privée** (ne la partagez JAMAIS !)
- `~/.ssh/tp-api-keypair.pub` → la **clé publique** (celle qu'on met sur le serveur)

**Sécurisez la clé privée :**

```bash
chmod 400 ~/.ssh/tp-api-keypair
```

> 💡 `chmod 400` = seul vous pouvez lire ce fichier. SSH refusera de fonctionner si d'autres personnes peuvent y accéder.

## Étape 1.4 : Trouver votre adresse IP publique

On va bientôt dire à AWS : "autorise la connexion SSH uniquement depuis **mon** ordinateur". Pour ça, il faut connaître votre IP publique.

```bash
curl ifconfig.me
# Exemple de résultat : 86.238.42.123
```

📝 **Notez cette adresse IP**, vous en aurez besoin à l'étape suivante.

> 💡 Si vous changez de réseau Wi-Fi, votre IP changera. Il faudra mettre à jour la configuration.

## Étape 1.5 : Créer les fichiers Terraform

Maintenant, on va écrire les fichiers qui décrivent notre serveur. Créez un dossier `terraform/` dans votre projet :

```bash
# À la racine de votre projet tp-api-nodejs
mkdir terraform
cd terraform
```

Vous allez créer **4 fichiers**. Voici à quoi sert chacun :

```
terraform/
├── main.tf           ← Le fichier principal : décrit le serveur, le pare-feu, etc.
├── variables.tf      ← Les paramètres modifiables (région, taille du serveur...)
├── outputs.tf        ← Les infos affichées après la création (IP du serveur, etc.)
├── terraform.tfvars  ← VOS valeurs personnelles (votre IP, chemins de fichiers...)
└── .gitignore        ← Les fichiers à ne PAS mettre sur GitHub
```

### Fichier 1 : `variables.tf` — Les paramètres

Ce fichier **déclare** les paramètres que notre configuration utilise. Pensez-y comme une liste d'ingrédients pour une recette.

Créez `terraform/variables.tf` :

```hcl
# ============================================
# VARIABLES — Les paramètres de notre infrastructure
# ============================================

# Dans quelle région AWS créer le serveur ?
variable "aws_region" {
  description = "Région AWS"
  type        = string
  default     = "us-east-1"   # N. Virginia — recommandé pour AWS Academy
}

# Quelle taille de serveur ?
variable "instance_type" {
  description = "Taille du serveur EC2"
  type        = string
  default     = "t2.micro"    # Le plus petit (gratuit avec Free Tier)
}

# Quel nom donner au serveur ?
variable "instance_name" {
  description = "Nom du serveur"
  type        = string
  default     = "tp-api-nodejs"
}

# Nom de la clé SSH dans AWS
variable "key_pair_name" {
  description = "Nom de la clé SSH"
  type        = string
  default     = "tp-api-keypair"
}

# Où se trouve votre clé publique SSH ?
variable "public_key_path" {
  description = "Chemin vers la clé publique SSH"
  type        = string
  default     = "~/.ssh/tp-api-keypair.pub"
}

# Où se trouve votre clé privée SSH ?
variable "private_key_path" {
  description = "Chemin vers la clé privée SSH"
  type        = string
  default     = "~/.ssh/tp-api-keypair"
}

# Sur quel port tourne votre API ?
variable "app_port" {
  description = "Port de l'API Node.js"
  type        = number
  default     = 3000
}

# Votre adresse IP (pour autoriser SSH)
variable "my_ip" {
  description = "Votre IP publique (format : x.x.x.x/32)"
  type        = string
  # Pas de valeur par défaut — vous devez la fournir
}
```

### Fichier 2 : `main.tf` — La description du serveur

C'est le cœur de la configuration. Il décrit **3 choses** :
1. **La clé SSH** pour se connecter au serveur
2. **Le pare-feu** (Security Group) : quels ports ouvrir
3. **Le serveur** (instance EC2) lui-même

Créez `terraform/main.tf` :

```hcl
# ============================================
# CONFIGURATION TERRAFORM
# ============================================
# Ce fichier décrit toute l'infrastructure :
# - Une clé SSH
# - Un pare-feu (Security Group)
# - Un serveur (instance EC2)
#
# Commandes :
#   terraform init    → Préparer le projet
#   terraform plan    → Voir ce qui va être créé
#   terraform apply   → Créer les ressources
#   terraform destroy → Tout supprimer
# ============================================

# ── Dire à Terraform qu'on utilise AWS ──
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  # Les identifiants AWS sont lus automatiquement depuis
  # les variables d'environnement (qu'on configurera à l'étape 1.6)
}

# ── Trouver automatiquement la bonne image de serveur ──
# Au lieu de chercher manuellement l'identifiant de l'image Amazon Linux,
# on demande à Terraform de trouver la plus récente pour nous.
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}

# ════════════════════════════════════════
# RESSOURCE 1 : La clé SSH
# ════════════════════════════════════════
# On envoie la clé PUBLIQUE à AWS.
# La clé PRIVÉE reste sur votre ordinateur.
resource "aws_key_pair" "tp_api_keypair" {
  key_name   = var.key_pair_name
  public_key = file(var.public_key_path)

  tags = {
    Name    = var.key_pair_name
    Project = "tp-api-nodejs"
  }
}

# ════════════════════════════════════════
# RESSOURCE 2 : Le pare-feu (Security Group)
# ════════════════════════════════════════
# Le pare-feu contrôle qui peut accéder au serveur et sur quels ports.
#
# Imaginez-le comme la porte d'entrée de votre serveur :
# - Port 22 (SSH)  : ouvert uniquement à VOTRE IP → vous seul pouvez entrer
# - Port 3000 (API): ouvert à tout le monde → tout le monde peut utiliser l'API
# - Port 80 (HTTP) : ouvert à tout le monde → accès web standard
resource "aws_security_group" "tp_api_sg" {
  name        = "tp-api-sg"
  description = "Pare-feu pour le TP API Node.js"

  # ── Qui peut ENTRER ? ──

  # Règle 1 : SSH — seulement depuis votre ordinateur
  ingress {
    description = "SSH depuis mon IP uniquement"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]
    # ⚠️ On ne met PAS 0.0.0.0/0 ici !
    # Sinon n'importe qui pourrait tenter de se connecter en SSH.
  }

  # Règle 2 : L'API Node.js — accessible par tout le monde
  ingress {
    description = "API Node.js (port 3000)"
    from_port   = var.app_port
    to_port     = var.app_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]    # 0.0.0.0/0 = tout Internet
  }

  # Règle 3 : HTTP standard
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # ── Qui peut SORTIR ? ──
  # Le serveur peut accéder à tout Internet (pour télécharger des paquets, etc.)
  egress {
    description = "Tout le trafic sortant"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"             # -1 = tous les protocoles
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name    = "tp-api-sg"
    Project = "tp-api-nodejs"
  }
}

# ════════════════════════════════════════
# RESSOURCE 3 : Le serveur (instance EC2)
# ════════════════════════════════════════
# C'est le serveur lui-même : une machine virtuelle dans le cloud.
# - t2.micro = 1 CPU, 1 Go de RAM (petit mais suffisant et gratuit)
resource "aws_instance" "tp_api" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  key_name               = aws_key_pair.tp_api_keypair.key_name
  vpc_security_group_ids = [aws_security_group.tp_api_sg.id]

  # Profil IAM pour AWS Academy
  iam_instance_profile = "LabInstanceProfile"

  tags = {
    Name        = var.instance_name
    Project     = "tp-api-nodejs"
    Environment = "production"
    ManagedBy   = "terraform"
  }

  # S'assurer que le pare-feu est créé AVANT le serveur
  depends_on = [aws_security_group.tp_api_sg]
}
```

### Fichier 3 : `outputs.tf` — Les infos utiles affichées à la fin

Après la création, Terraform affichera automatiquement les informations dont vous avez besoin (IP du serveur, commande SSH, URL de l'API...).

Créez `terraform/outputs.tf` :

```hcl
# ============================================
# OUTPUTS — Infos affichées après terraform apply
# ============================================

output "instance_public_ip" {
  description = "Adresse IP publique du serveur"
  value       = aws_instance.tp_api.public_ip
}

output "api_url" {
  description = "URL de votre API"
  value       = "http://${aws_instance.tp_api.public_ip}:${var.app_port}"
}

output "ssh_command" {
  description = "Commande pour se connecter en SSH"
  value       = "ssh -i ${var.private_key_path} ec2-user@${aws_instance.tp_api.public_ip}"
}

output "instance_id" {
  description = "Identifiant du serveur EC2"
  value       = aws_instance.tp_api.id
}
```

### Fichier 4 : `terraform.tfvars` — Vos valeurs personnelles

Ce fichier contient les valeurs spécifiques à **votre** environnement.

Créez `terraform/terraform.tfvars` :

```hcl
# ============================================
# VOS VALEURS PERSONNELLES
# ============================================
# ⚠️ Ce fichier ne sera PAS commité sur GitHub (il est dans .gitignore)

aws_region       = "us-east-1"
instance_type    = "t2.micro"
instance_name    = "tp-api-nodejs"
key_pair_name    = "tp-api-keypair"
public_key_path  = "~/.ssh/tp-api-keypair.pub"
private_key_path = "~/.ssh/tp-api-keypair"
app_port         = 3000

# ⚠️ REMPLACEZ par votre vraie IP (trouvée à l'étape 1.4)
# N'oubliez pas le /32 à la fin !
my_ip = "VOTRE_IP_ICI/32"
```

> ⚠️ **IMPORTANT** : Remplacez `VOTRE_IP_ICI` par l'IP que vous avez trouvée à l'étape 1.4.
> Exemple : `my_ip = "86.238.42.123/32"`

### Fichier 5 : `.gitignore` — Ne pas mettre certains fichiers sur GitHub

Créez `terraform/.gitignore` :

```gitignore
# L'état de Terraform (contient des infos sensibles)
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl

# Vos valeurs personnelles (contient votre IP)
terraform.tfvars

# Logs de crash
crash.log
crash.*.log

# Plans sauvegardés
*.tfplan
```

## Étape 1.6 : Donner à Terraform vos identifiants AWS

Terraform a besoin de vos identifiants AWS pour créer des ressources. Ces identifiants viennent de Vocareum.

**1. Récupérez vos identifiants :**

Dans Vocareum, cliquez sur **AWS Details**, puis sur **Show** à côté de "AWS CLI". Vous verrez quelque chose comme :

```
[default]
aws_access_key_id=ASIA...
aws_secret_access_key=abc123...
aws_session_token=FwoGZX...
```

**2. Exportez-les dans votre terminal :**

```bash
export AWS_ACCESS_KEY_ID="copiez_la_valeur_ici"
export AWS_SECRET_ACCESS_KEY="copiez_la_valeur_ici"
export AWS_SESSION_TOKEN="copiez_la_valeur_ici"
```

> ⚠️ **À refaire à chaque fois** que vous relancez le lab ou que les identifiants expirent.

**3. Vérifiez que ça marche :**

```bash
aws sts get-caller-identity
```

Si vous voyez un JSON avec votre Account et votre ARN, c'est bon ! ✅

> 💡 Si la commande `aws` n'est pas trouvée, ce n'est pas grave : Terraform utilisera quand même les variables d'environnement.

## Étape 1.7 : Créer le serveur !

C'est le moment de vérité. On va exécuter **3 commandes** dans l'ordre :

```bash
cd terraform
```

### Commande 1 : `terraform init` — Préparer le projet

```bash
terraform init
```

Cette commande télécharge le plugin AWS. Vous devriez voir :

```
Terraform has been successfully initialized! ✅
```

### Commande 2 : `terraform plan` — Voir ce qui va être créé

```bash
terraform plan
```

Terraform vous montre **exactement** ce qu'il va faire, **sans rien créer**. C'est comme un devis avant des travaux.

```
Plan: 3 to add, 0 to change, 0 to destroy.
```

> 💡 **Lisez toujours le plan.** Il vous dit :
> - `+` = ressource à créer
> - `~` = ressource à modifier
> - `-` = ressource à supprimer
>
> Ici, 3 ressources seront créées : la clé SSH, le pare-feu, et le serveur.

### Commande 3 : `terraform apply` — Créer les ressources

```bash
terraform apply
```

Terraform vous montre encore le plan, puis demande confirmation :

```
Do you want to perform these actions?
  Enter a value: yes
```

**Tapez `yes` et appuyez sur Entrée.**

Attendez environ 30 secondes... et vous verrez :

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed. ✅

Outputs:

instance_public_ip = "54.172.xxx.xxx"
api_url            = "http://54.172.xxx.xxx:3000"
ssh_command        = "ssh -i ~/.ssh/tp-api-keypair ec2-user@54.172.xxx.xxx"
```

🎉 **Votre serveur existe !** Terraform vous donne directement l'IP et la commande SSH.

> 💡 **Astuce** : Vous pouvez revoir ces infos à tout moment avec :
> ```bash
> terraform output
> ```

## Étape 1.8 : Se connecter au serveur en SSH

Copiez la commande SSH affichée par Terraform et exécutez-la :

```bash
ssh -i ~/.ssh/tp-api-keypair ec2-user@VOTRE_IP_PUBLIQUE
```

> 💡 Remplacez `VOTRE_IP_PUBLIQUE` par l'IP affichée dans les outputs. Ou utilisez directement :
> ```bash
> ssh -i ~/.ssh/tp-api-keypair ec2-user@$(terraform output -raw instance_public_ip)
> ```

Si tout va bien, vous voyez l'écran de bienvenue Amazon Linux :

```
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
[ec2-user@ip-xxx-xxx-xxx-xxx ~]$
```

**Vous êtes connecté à votre serveur dans le cloud !** 🎉

Tapez `exit` pour revenir sur votre machine.

## Étape 1.9 : Vérifier l'idempotence de Terraform

Relancez `terraform apply` **sans rien changer** :

```bash
terraform apply
```

Résultat :

```
No changes. Your infrastructure matches the configuration. ✅
```

Terraform compare ce qui existe déjà avec ce que vous avez demandé, et ne fait **rien** si tout est déjà en place. C'est ce qu'on appelle l'**idempotence** : appliquer 2 fois = même résultat.

## Étape 1.10 : Sauvegarder votre code Terraform

```bash
cd ..  # Retour à la racine du projet

git add terraform/main.tf terraform/variables.tf terraform/outputs.tf terraform/.gitignore
git commit -m "infra: création du serveur EC2 avec Terraform"
git push origin main
```

> 💡 On ne commite **pas** `terraform.tfvars` (contient votre IP) ni les fichiers `.tfstate` (gérés par le `.gitignore`).

**❓ Questions de compréhension :**
1. Quelle est la différence entre `terraform plan` et `terraform apply` ?
2. Pourquoi n'ouvre-t-on pas le port SSH (22) à `0.0.0.0/0` (tout Internet) ?
3. Que se passe-t-il si quelqu'un obtient votre clé privée SSH ?
4. Pourquoi le fichier `terraform.tfstate` ne doit-il pas être mis sur GitHub ?

---

# 🔧 PARTIE 2 : Configurer le Serveur Manuellement (pour comprendre)

Votre serveur est une **machine vierge**. Il n'y a rien dessus — pas de Node.js, pas de PM2, rien. Avant d'automatiser avec Ansible, on va tout faire à la main une première fois pour **comprendre ce qu'on automatise**.

> 💡 C'est comme apprendre à faire la cuisine avant d'acheter un robot : il faut d'abord comprendre les étapes.

## Étape 2.1 : Se connecter au serveur

```bash
ssh -i ~/.ssh/tp-api-keypair ec2-user@$(cd terraform && terraform output -raw instance_public_ip)
```

Vous êtes maintenant **sur le serveur** (pas sur votre machine).

## Étape 2.2 : Installer Node.js

```bash
# 1. Mettre à jour le système (comme "apt update" sur Ubuntu)
sudo yum update -y

# 2. Ajouter le dépôt de Node.js 20
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -

# 3. Installer Node.js
sudo yum install -y nodejs

# 4. Vérifier que ça a marché
node --version    # → v20.x.x
npm --version     # → 10.x.x
```

## Étape 2.3 : Installer PM2

**PM2** garde votre application en vie, même quand vous fermez le terminal.

Sans PM2 : vous lancez `node server.js`, vous fermez le terminal → l'application s'arrête.
Avec PM2 : l'application tourne en arrière-plan, redémarre automatiquement si elle plante.

```bash
sudo npm install -g pm2
pm2 --version
```

## Étape 2.4 : Installer Git et préparer le dossier

```bash
sudo yum install -y git
mkdir -p /home/ec2-user/app
```

## Étape 2.5 : Tester un déploiement à la main

```bash
cd /home/ec2-user/app

# Cloner votre dépôt (remplacez VOTRE_USERNAME)
git clone https://github.com/VOTRE_USERNAME/tp-api-nodejs.git .

# Installer les dépendances (sans les outils de développement)
npm install --omit=dev

# Configurer l'application
echo "PORT=3000" > .env
echo "NODE_ENV=production" >> .env
echo "USE_MEMORY_DB=true" >> .env

# Lancer l'application avec PM2
pm2 start server.js --name "tp-api"

# Vérifier que ça tourne
pm2 status
```

Maintenant, ouvrez votre navigateur et allez sur :

```
http://VOTRE_IP_PUBLIQUE:3000
```

> 💡 Retrouvez l'IP avec `terraform output -raw instance_public_ip` depuis votre machine locale.

**Si vous voyez une réponse JSON, ça marche !** 🎉

Nettoyons avant de passer à l'automatisation :

```bash
pm2 delete tp-api
rm -rf /home/ec2-user/app/*
exit    # Revenir sur votre machine
```

**❓ Question :** Pourquoi installe-t-on les dépendances avec `--omit=dev` en production ?

---

# 🚀 PARTIE 3 : Déploiement Automatique avec GitHub Actions

## Ce qu'on va faire

Remplacer le déploiement simulé par un **vrai déploiement** sur votre serveur EC2.

```
Avant (simulé)                 Maintenant (réel)
──────────────                 ─────────────────
echo "Déployé !"        →     1. Se connecte en SSH au serveur
                               2. Copie les fichiers
                               3. Installe les dépendances
                               4. Redémarre l'application
                               5. Vérifie que ça marche ✅
```

## Étape 3.1 : Ajouter les secrets dans GitHub

GitHub Actions a besoin de 3 informations pour se connecter à votre serveur. On les stocke en tant que **secrets** (chiffrés et invisibles dans les logs).

1. Allez dans votre dépôt GitHub → **Settings** → **Secrets and variables** → **Actions**
2. Cliquez sur **New repository secret** pour chaque secret :

| Nom du secret | Quoi mettre dedans |
|---|---|
| `EC2_SSH_KEY` | Le **contenu** de votre clé privée (voir commande ci-dessous) |
| `EC2_HOST` | L'IP publique de votre serveur (ex: `54.172.xxx.xxx`) |
| `EC2_USER` | `ec2-user` |

**Pour copier le contenu de la clé privée :**

```bash
cat ~/.ssh/tp-api-keypair
```

Copiez **TOUT** ce qui s'affiche, depuis `-----BEGIN` jusqu'à `-----END...-----`.

**Pour récupérer l'IP :**

```bash
cd terraform && terraform output -raw instance_public_ip
```

## Étape 3.2 : Créer le fichier de configuration PM2

PM2 a besoin d'un fichier de configuration pour savoir comment lancer votre application.

Créez `ecosystem.config.js` **à la racine de votre projet** :

```javascript
module.exports = {
  apps: [{
    name: 'tp-api',
    script: './server.js',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '200M',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
};
```

```bash
git add ecosystem.config.js
git commit -m "chore: ajout de la configuration PM2"
git push origin main
```

## Étape 3.3 : Créer le workflow de déploiement

Remplacez le contenu de `.github/workflows/deploy.yml` :

```yaml
name: CD - Déploiement AWS

on:
  # Se déclenche quand le workflow CI réussit sur main
  workflow_run:
    workflows: ["CI - Tests"]
    types:
      - completed
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: 🚀 Déployer sur EC2

    # Seulement si le CI a réussi
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    environment:
      name: production

    steps:
      # 1. Récupérer le code
      - name: 📥 Checkout du code
        uses: actions/checkout@v4

      # 2. Préparer la connexion SSH
      - name: 🔑 Configurer la clé SSH
        run: |
          mkdir -p ~/.ssh
          printf "%s" "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/ec2_key
          chmod 600 ~/.ssh/ec2_key
          ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts 2>/dev/null || true

      # 3. Copier les fichiers sur le serveur
      - name: 📦 Envoyer les fichiers au serveur
        run: |
          rsync -avz --delete \
            --exclude '.git' \
            --exclude 'node_modules' \
            --exclude '.env' \
            --exclude 'coverage' \
            --exclude '__tests__' \
            --exclude 'terraform' \
            -e "ssh -i ~/.ssh/ec2_key" \
            ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/home/${{ secrets.EC2_USER }}/app/

      # 4. Installer les dépendances et redémarrer
      - name: 🔄 Installer et redémarrer l'application
        run: |
          ssh -o StrictHostKeyChecking=no -o ServerAliveInterval=60 \
            -i ~/.ssh/ec2_key \
            ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'ENDSSH'
              cd /home/ec2-user/app
              echo "📦 Installation des dépendances..."
              npm install --omit=dev
              echo "🔄 Redémarrage de l'application..."
              pm2 delete tp-api || true
              pm2 start ecosystem.config.js --env production
              sleep 3
              pm2 status
          ENDSSH

      # 5. Vérifier que l'API répond
      - name: 🏥 Vérification de santé
        run: |
          echo "🔍 Vérification que l'API répond..."
          for i in {1..10}; do
            HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
              http://${{ secrets.EC2_HOST }}:3000/ || echo "000")
            if [ "$HTTP_STATUS" -ge 200 ] && [ "$HTTP_STATUS" -lt 500 ]; then
              echo "✅ API accessible ! (HTTP $HTTP_STATUS)"
              exit 0
            fi
            echo "⏳ Tentative $i/10 — HTTP $HTTP_STATUS — nouvelle tentative dans 3s..."
            sleep 3
          done
          echo "❌ L'API ne répond pas après 30 secondes"
          exit 1

      # 6. Nettoyer la clé SSH
      - name: 🧹 Nettoyage
        if: always()
        run: rm -f ~/.ssh/ec2_key
```

> 💡 **Qu'est-ce que `rsync` ?** C'est un outil qui copie des fichiers via SSH, mais de manière **intelligente** : il ne copie que les fichiers qui ont changé. Beaucoup plus rapide que de tout recopier à chaque fois.

```bash
git add .github/workflows/deploy.yml
git commit -m "cd: déploiement réel sur AWS EC2"
git push origin main
```

## Étape 3.4 : Observer le déploiement

1. Allez dans l'onglet **Actions** de votre dépôt GitHub
2. Le workflow **CI - Tests** se lance
3. Quand il passe ✅, le workflow **CD - Déploiement AWS** se déclenche automatiquement
4. Cliquez dessus pour voir chaque étape en temps réel
5. À la fin, ouvrez `http://VOTRE_IP_PUBLIQUE:3000` dans votre navigateur

**Votre API est en ligne sur Internet !** 🌍🎉

**❓ Questions :**
1. Pourquoi utilise-t-on `rsync` plutôt que de simplement copier tous les fichiers ?
2. À quoi sert la vérification de santé (health check) ?

---

# 🔁 PARTIE 4 : Tester que Tout Fonctionne de Bout en Bout

## Étape 4.1 : Ajouter une nouvelle route et voir le déploiement automatique

On va prouver que le déploiement est vraiment automatique en ajoutant une nouvelle route.

```bash
git checkout -b feature/health-route
```

Ajoutez cette route dans votre fichier principal (`app.js` ou `routes/`) :

```javascript
// Route pour vérifier que l'API fonctionne
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    environment: process.env.NODE_ENV || 'development',
    version: require('./package.json').version
  });
});
```

Ajoutez un test :

```javascript
describe('GET /health', () => {
  it('devrait retourner le statut de santé', async () => {
    const response = await request(app).get('/health');
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('status', 'ok');
    expect(response.body).toHaveProperty('uptime');
    expect(response.body).toHaveProperty('timestamp');
  });
});
```

```bash
npm test                          # Vérifier localement
git add .
git commit -m "feat: route /health avec test"
git push origin feature/health-route
```

Créez une **Pull Request** sur GitHub, puis :

1. ✅ CI passe → les tests sont verts
2. Fusionnez la PR dans `main`
3. CI relancée sur `main` → ✅
4. CD se déclenche → déploiement automatique sur EC2
5. Ouvrez `http://VOTRE_IP_PUBLIQUE:3000/health`

Vous voyez en direct depuis votre serveur AWS :

```json
{
  "status": "ok",
  "timestamp": "2026-04-13T14:30:00.000Z",
  "uptime": 42.5,
  "environment": "production",
  "version": "1.0.0"
}
```

🎉 **Le cycle est complet : push → tests → déploiement automatique !**

## Étape 4.2 : Vérifier que le code cassé ne passe PAS en production

```bash
git checkout -b feature/test-echec
```

Ajoutez volontairement une erreur dans `server.js` :

```javascript
throw new Error('Erreur intentionnelle pour tester le pipeline');
```

```bash
git add server.js
git commit -m "test: erreur intentionnelle"
git push origin feature/test-echec
```

Créez une PR. Le **CI** échoue (les tests ne passent pas) → le **CD ne se déclenche jamais**.

**Le code cassé n'atteint jamais la production.** C'est le but de tout ce pipeline !

Fermez la PR sans fusionner et supprimez la branche.

**❓ Question :** Si le déploiement échoue (ex: le serveur est éteint), que devrait faire l'équipe ?

---

# 🤖 PARTIE 5 : Introduction à Ansible

## Le problème

Dans la Partie 2, vous avez configuré le serveur **à la main** : `yum update`, `curl | bash`, `npm install -g pm2`...

C'est fastidieux et fragile :

| Problème | Exemple concret |
|----------|----------------|
| **Pas reproductible** | Votre serveur plante → vous avez oublié les commandes exactes |
| **Pas versionné** | Impossible de savoir qui a changé quoi sur le serveur |
| **Pas scalable** | 10 serveurs = taper les mêmes commandes 10 fois |
| **Sujet aux erreurs** | Une faute de frappe et tout casse |

## La solution : Ansible

**Ansible** est un outil qui automatise la configuration des serveurs. Au lieu de taper des commandes, vous écrivez un fichier YAML qui décrit **ce que le serveur doit avoir**.

```
Configuration manuelle               Ansible
──────────────────────               ────────
"J'ai installé Node sur le serveur"  "Le serveur DOIT avoir Node 20"
→ Non reproductible                  → 100% reproductible
→ "Ça marchait quand je l'ai fait"   → Même résultat à chaque exécution
```

**L'idée clé :** Ansible est **idempotent**. Ça veut dire que vous pouvez l'exécuter 10 fois, et il fera toujours la même chose : il vérifie l'état actuel et ne fait **que ce qui manque**.

```
Première exécution : "Node.js pas installé → je l'installe"     → changed
Deuxième exécution : "Node.js déjà installé → rien à faire"     → ok
```

## Terraform vs Ansible — Qui fait quoi ?

```
TERRAFORM = crée les machines           ANSIBLE = configure les machines
         (le "hardware")                        (le "software")

┌──────────────────────┐               ┌───────────────────────┐
│  ✅ Créer le serveur  │               │  ✅ Installer Node.js  │
│  ✅ Créer le pare-feu │    ───→       │  ✅ Installer PM2      │
│  ✅ Créer la clé SSH  │   IP du       │  ✅ Déployer le code   │
│                       │   serveur     │  ✅ Configurer .env    │
└──────────────────────┘               └───────────────────────┘

"Quelles machines existent ?"          "Que fait la machine ?"
```

## Comment Ansible fonctionne

Ansible se connecte à vos serveurs **via SSH** (pas besoin d'installer quoi que ce soit sur le serveur) et exécute les tâches que vous avez décrites.

```
Votre machine                   Serveur EC2
(vous lancez Ansible ici)       (Ansible s'y connecte via SSH)
────────────────────            ──────────────────────────────
ansible-playbook     ── SSH →   Installe Node.js ✅
  playbook.yml                  Installe PM2 ✅
                                Clone le code ✅
                                Démarre l'app ✅
                     ← Rapport  ok=10  changed=4  failed=0
```

**Vocabulaire Ansible :**

| Terme | Explication simple |
|-------|-------------------|
| **Playbook** | Un fichier YAML avec la liste des tâches à faire |
| **Task** | Une action (ex: "installer Git") |
| **Module** | L'outil qu'Ansible utilise pour faire l'action (ex: `yum` pour installer des paquets) |
| **Inventory** | La liste de vos serveurs et comment s'y connecter |
| **Handler** | Une action qui se déclenche seulement si quelque chose a changé |
| **Role** | Un "paquet" réutilisable de tâches (comme une fonction en programmation) |

**❓ Questions :**
1. Que signifie "agentless" (sans agent) et pourquoi est-ce pratique ?
2. Que signifie "idempotent" ? Donnez un exemple.
3. Quelle est la différence entre Ansible et un script shell ?

---

# 📦 PARTIE 6 : Installer et Configurer Ansible

## Étape 6.1 : Installer Ansible

Ansible s'installe **sur votre machine** (pas sur le serveur).

<details>
<summary>🍎 macOS</summary>

```bash
brew install ansible
```
</details>

<details>
<summary>🐧 Linux (Ubuntu/Debian)</summary>

```bash
sudo apt update
sudo apt install -y ansible
```
</details>

<details>
<summary>🐧 Linux (Fedora/RHEL/Amazon Linux)</summary>

```bash
sudo pip3 install ansible
```
</details>

<details>
<summary>🪟 Windows</summary>

Utilisez WSL (Windows Subsystem for Linux), puis suivez les instructions Linux.
</details>

**Vérification :**

```bash
ansible --version
# → ansible [core 2.16.x]
```

## Étape 6.2 : Créer les fichiers Ansible

```bash
# À la racine de votre projet
mkdir -p ansible/templates
cd ansible
```

Voici ce qu'on va créer :

```
ansible/
├── ansible.cfg           ← Configuration d'Ansible
├── inventory.ini         ← Liste des serveurs
├── playbook-setup.yml    ← Installe Node.js, PM2, Git sur le serveur
├── playbook-deploy.yml   ← Déploie l'application
└── templates/
    └── env.j2            ← Modèle pour le fichier .env
```

## Étape 6.3 : Le fichier d'inventaire (`inventory.ini`)

Ce fichier dit à Ansible **quels serveurs gérer** et **comment s'y connecter**.

Créez `ansible/inventory.ini` :

```ini
# ============================================
# LISTE DES SERVEURS
# ============================================

[webservers]
# Remplacez l'IP par celle de votre serveur
# (cf. terraform output -raw instance_public_ip)
ec2-api ansible_host=VOTRE_IP_PUBLIQUE

[webservers:vars]
# Comment se connecter
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/tp-api-keypair
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o ConnectTimeout=15'
ansible_ssh_timeout=20
```

> ⚠️ **Remplacez `VOTRE_IP_PUBLIQUE`** par la vraie IP de votre serveur.

## Étape 6.4 : Le fichier de configuration (`ansible.cfg`)

Créez `ansible/ansible.cfg` :

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False
force_color = True
timeout = 30

[privilege_escalation]
become = False
become_method = sudo
become_ask_pass = False
```

## Étape 6.5 : Tester la connexion

```bash
cd ansible
ansible webservers -m ping
```

Résultat attendu :

```
ec2-api | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

**Si vous voyez `pong`, Ansible peut se connecter à votre serveur !** 🎉

> ⚠️ **Si ça ne marche pas**, vérifiez :
> - L'IP dans `inventory.ini` est correcte
> - Le chemin `~/.ssh/tp-api-keypair` existe
> - Le serveur EC2 est en état **Running**
> - Le port 22 est ouvert dans le Security Group pour votre IP

**❓ Question :** Pourquoi Ansible a-t-il besoin de Python sur le serveur, s'il est "sans agent" ?

---

# 📝 PARTIE 7 : Le Playbook de Configuration du Serveur

Ce playbook installe tout ce qu'il faut sur un serveur vierge : Node.js, PM2, Git.

## Étape 7.1 : Écrire le playbook

Créez `ansible/playbook-setup.yml` :

```yaml
# ============================================
# PLAYBOOK : Installer les logiciels sur le serveur
# ============================================
# Ce playbook prend un serveur vierge et installe :
# - Node.js 20
# - PM2 (gestionnaire de processus)
# - Git
#
# Lancer : ansible-playbook playbook-setup.yml
# ============================================

---
- name: 🔧 Configuration du serveur
  hosts: webservers        # Sur quels serveurs ? (définis dans inventory.ini)
  become: yes              # Utiliser sudo (nécessaire pour installer des paquets)

  # ── Les paramètres ──
  vars:
    node_version: "20"
    app_user: "ec2-user"
    app_dir: "/home/ec2-user/app"
    app_port: 3000

  # ── Les tâches à effectuer ──
  tasks:

    # 1. Mettre à jour le système
    - name: 📦 Mettre à jour le système
      yum:
        name: "*"
        state: latest

    # 2. Installer les outils de base
    - name: 🔧 Installer Git, wget et unzip
      yum:
        name:
          - git
          - wget
          - unzip
        state: present
      # state: present = "installe si pas déjà installé"
      # C'est l'idempotence : si c'est déjà là, Ansible ne fait rien

    # 3. Ajouter le dépôt Node.js
    - name: 🟢 Configurer le dépôt Node.js {{ node_version }}
      shell: |
        curl -fsSL https://rpm.nodesource.com/setup_{{ node_version }}.x | bash -
      args:
        creates: /etc/yum.repos.d/nodesource-el9.repo
      # "creates" = si ce fichier existe déjà, sauter cette étape

    # 4. Installer Node.js
    - name: 🟢 Installer Node.js
      yum:
        name: nodejs
        state: present

    # 5. Vérifier Node.js
    - name: 🟢 Vérifier la version de Node.js
      command: node --version
      register: node_version_output    # Sauvegarder le résultat
      changed_when: false              # Cette tâche ne "change" rien

    - name: 📋 Afficher la version de Node.js
      debug:
        msg: "Node.js installé : {{ node_version_output.stdout }}"

    # 6. Installer PM2
    - name: 📦 Installer PM2
      npm:
        name: pm2
        global: yes
        state: present

    - name: 📋 Vérifier PM2
      command: pm2 --version
      register: pm2_version_output
      changed_when: false

    - name: 📋 Afficher la version de PM2
      debug:
        msg: "PM2 installé : {{ pm2_version_output.stdout }}"

    # 7. Créer le dossier de l'application
    - name: 📁 Créer le dossier /home/ec2-user/app
      file:
        path: "{{ app_dir }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'

    # 8. Configurer PM2 pour redémarrer au boot
    - name: 🔄 Configurer PM2 au démarrage du serveur
      command: "pm2 startup systemd -u {{ app_user }} --hp /home/{{ app_user }}"
      become_user: root
      register: pm2_startup
      changed_when: "'already' not in pm2_startup.stdout"

    # 9. Ouvrir le port dans le pare-feu (si actif)
    - name: 🔥 Vérifier si le pare-feu est actif
      command: systemctl is-active firewalld
      register: firewalld_status
      changed_when: false
      failed_when: false

    - name: 🔥 Ouvrir le port {{ app_port }}
      firewalld:
        port: "{{ app_port }}/tcp"
        permanent: yes
        state: enabled
        immediate: yes
      when: firewalld_status.rc == 0
      # "when" = exécuter cette tâche SEULEMENT si la condition est vraie

  handlers:
    - name: Redémarrer PM2
      become_user: "{{ app_user }}"
      command: pm2 reload all
```

## Étape 7.2 : Lancer le playbook

```bash
ansible-playbook playbook-setup.yml
```

Regardez la progression. Chaque tâche affiche un résultat :

- **ok** (vert) = déjà fait, rien à changer
- **changed** (jaune) = quelque chose a été modifié
- **failed** (rouge) = erreur
- **skipped** (bleu) = tâche ignorée (condition `when` non remplie)

À la fin, vous voyez le **résumé** :

```
PLAY RECAP ************************************************************
ec2-api : ok=12   changed=7    unreachable=0    failed=0    skipped=1
```

## Étape 7.3 : Vérifier l'idempotence

Relancez **exactement la même commande** :

```bash
ansible-playbook playbook-setup.yml
```

Cette fois :

```
PLAY RECAP ************************************************************
ec2-api : ok=12   changed=0    unreachable=0    failed=0    skipped=1
```

**`changed=0`** ! Tout est déjà installé, Ansible n'a rien fait. C'est l'idempotence en action.

**❓ Questions :**
1. Quelle est la différence entre `ok` et `changed` ?
2. Pourquoi c'est important que `changed=0` au deuxième lancement ?
3. À quoi sert `become: yes` ?

---

# 🚀 PARTIE 8 : Le Playbook de Déploiement

## Étape 8.1 : Créer le template du fichier `.env`

Créez `ansible/templates/env.j2` :

```jinja2
# Fichier généré par Ansible — ne pas modifier à la main !
# Dernière mise à jour : {{ ansible_date_time.iso8601 if ansible_date_time is defined else lookup('pipe', 'date -Iseconds') }}

PORT={{ app_port }}
NODE_ENV={{ node_env }}
{% if mongodb_uri is defined and mongodb_uri != '' %}
MONGODB_URI={{ mongodb_uri }}
{% endif %}
```

> 💡 C'est un **template** : les `{{ variable }}` sont remplacées par les vraies valeurs quand Ansible l'exécute. Le `.j2` signifie qu'il utilise le moteur de templates **Jinja2**.

## Étape 8.2 : Écrire le playbook de déploiement

Créez `ansible/playbook-deploy.yml` :

```yaml
# ============================================
# PLAYBOOK : Déployer l'application
# ============================================
# Ce playbook récupère le code depuis GitHub,
# installe les dépendances, et lance l'application.
#
# Prérequis : avoir lancé playbook-setup.yml une première fois
# Lancer : ansible-playbook playbook-deploy.yml
# ============================================

---
- name: 🚀 Déploiement de l'API Node.js
  hosts: webservers
  become: no              # On déploie en tant que ec2-user (pas root)
  gather_facts: false

  vars:
    app_dir: "/home/ec2-user/app"
    app_port: 3000
    node_env: "production"
    github_repo: "https://github.com/VOTRE_USERNAME/tp-api-nodejs.git"
    github_branch: "main"

  pre_tasks:
    - name: 🔌 Vérifier que le serveur est joignable
      wait_for_connection:
        timeout: 25
        connect_timeout: 8
        sleep: 2

  tasks:

    # ── 1. Récupérer le code ──

    - name: 📁 S'assurer que le dossier existe
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: 🔎 Vérifier si le code est déjà cloné
      stat:
        path: "{{ app_dir }}/.git"
      register: repo_git_dir

    - name: 📥 Cloner le dépôt (première fois)
      shell: |
        timeout 120 git clone --depth 1 --branch {{ github_branch }} {{ github_repo }} {{ app_dir }}
      environment:
        GIT_TERMINAL_PROMPT: "0"
      when: not repo_git_dir.stat.exists

    - name: 🔄 Mettre à jour le code (fois suivantes)
      shell: |
        timeout 120 git fetch --depth 1 origin {{ github_branch }}
        git checkout {{ github_branch }}
        git reset --hard origin/{{ github_branch }}
      args:
        chdir: "{{ app_dir }}"
      environment:
        GIT_TERMINAL_PROMPT: "0"
      when: repo_git_dir.stat.exists

    - name: 🧾 Quel commit est déployé ?
      command: git rev-parse HEAD
      args:
        chdir: "{{ app_dir }}"
      register: deployed_commit
      changed_when: false

    - name: 📋 Afficher le commit
      debug:
        msg: "Commit déployé : {{ deployed_commit.stdout }}"

    # ── 2. Configurer l'application ──

    - name: ⚙️ Générer le fichier .env
      template:
        src: templates/env.j2
        dest: "{{ app_dir }}/.env"
        mode: '0600'       # Seul le propriétaire peut lire ce fichier

    # ── 3. Installer les dépendances ──

    - name: 📦 Installer les dépendances npm (production)
      npm:
        path: "{{ app_dir }}"
        production: yes
        state: present

    # ── 4. Vérifier que le fichier PM2 existe ──

    - name: 📋 Vérifier ecosystem.config.js
      stat:
        path: "{{ app_dir }}/ecosystem.config.js"
      register: ecosystem_file

    - name: ❌ Erreur si ecosystem.config.js manque
      fail:
        msg: "Le fichier ecosystem.config.js n'existe pas ! Ajoutez-le à votre dépôt Git."
      when: not ecosystem_file.stat.exists

    # ── 5. Lancer l'application ──

    - name: 🚀 Démarrer/Redémarrer l'application
      shell: |
        timeout 60 pm2 startOrReload ecosystem.config.js --env production
      args:
        chdir: "{{ app_dir }}"
      register: pm2_result
      changed_when: true
      failed_when: pm2_result.rc != 0

    - name: 📋 Statut PM2
      command: pm2 jlist
      register: pm2_status
      changed_when: false

    - name: 📋 Afficher le statut
      debug:
        msg: "{{ pm2_status.stdout | from_json | map(attribute='name') | list }} — en cours d'exécution"
      ignore_errors: yes

    - name: 💾 Sauvegarder la config PM2 (survit au redémarrage)
      command: pm2 save
      changed_when: false

    # ── 6. Vérifier que l'API marche ──

    - name: ⏳ Attendre 3 secondes que l'app démarre
      pause:
        seconds: 3

    - name: 🏥 Vérifier que l'API répond
      uri:
        url: "http://localhost:{{ app_port }}/"
        method: GET
        status_code: [200, 301, 302, 404]
        timeout: 10
      register: health_check
      retries: 5
      delay: 3
      until: health_check.status != -1

    - name: ✅ Déploiement réussi !
      debug:
        msg: |
          ════════════════════════════════════════
           ✅ DÉPLOIEMENT RÉUSSI !
          ════════════════════════════════════════
           🌐 URL : http://{{ ansible_host }}:{{ app_port }}
           📦 Commit : {{ deployed_commit.stdout[:8] }}
           📅 Date : {{ lookup('pipe', 'date -Iseconds') }}
          ════════════════════════════════════════
```

## Étape 8.3 : Lancer le déploiement

⚠️ **Avant de lancer**, modifiez la variable `github_repo` dans le fichier : remplacez `VOTRE_USERNAME` par votre vrai nom d'utilisateur GitHub.

```bash
ansible-playbook playbook-deploy.yml
```

À la fin :

```
TASK [✅ Déploiement réussi !]
ok: [ec2-api] => {
    "msg": "════════════════════════════════════════\n ✅ DÉPLOIEMENT RÉUSSI !\n 🌐 URL : http://54.xxx.xxx.xxx:3000\n ..."
}
```

Ouvrez l'URL dans votre navigateur. **Votre API est déployée par Ansible !** 🎉

**❓ Questions :**
1. Quelle est la différence entre `playbook-setup` et `playbook-deploy` ?
2. Pourquoi utilise-t-on un template pour `.env` au lieu de copier un fichier ?
3. Que se passe-t-il si le health check échoue 5 fois ?

---

# 🔗 PARTIE 9 : Combiner Ansible + GitHub Actions

Maintenant, on va remplacer les commandes SSH manuelles dans le workflow GitHub Actions par **Ansible**. C'est plus structuré et maintenable.

```
GitHub Actions                   Ansible                    EC2
──────────────                   ────────                   ────
1. Tests passent ✅
2. Installe Ansible  ──────→     3. playbook-deploy.yml     4. App mise à jour
                                    - git pull              5. ✅ En ligne !
                                    - npm install
                                    - pm2 reload
                                    - health check
```

## Étape 9.1 : Créer le workflow

Remplacez `.github/workflows/deploy.yml` :

```yaml
name: CD - Déploiement AWS (Ansible)

on:
  workflow_run:
    workflows: ["CI - Tests"]
    types:
      - completed
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: 🚀 Déployer avec Ansible

    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    environment:
      name: production

    steps:
      - name: 📥 Checkout du code
        uses: actions/checkout@v4

      - name: 🤖 Installer Ansible
        run: |
          sudo apt-get update
          sudo apt-get install -y ansible
          ansible --version

      - name: 🔑 Configurer SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/ec2_key
          chmod 600 ~/.ssh/ec2_key
          ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts 2>/dev/null

      - name: 📋 Créer l'inventaire Ansible
        run: |
          cat > ansible/inventory-ci.ini << EOF
          [webservers]
          ec2-api ansible_host=${{ secrets.EC2_HOST }}

          [webservers:vars]
          ansible_user=${{ secrets.EC2_USER }}
          ansible_ssh_private_key_file=~/.ssh/ec2_key
          ansible_ssh_common_args='-o StrictHostKeyChecking=no'
          EOF

      - name: 🏓 Tester la connexion
        run: |
          cd ansible
          ansible webservers -i inventory-ci.ini -m ping

      - name: 🚀 Déployer avec Ansible
        run: |
          cd ansible
          ansible-playbook -i inventory-ci.ini playbook-deploy.yml \
            -e "github_repo=https://github.com/${{ github.repository }}.git" \
            -e "github_branch=${{ github.ref_name }}"

      - name: 🏥 Vérification de santé finale
        run: |
          for i in {1..10}; do
            HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
              http://${{ secrets.EC2_HOST }}:3000/ || echo "000")
            if [ "$HTTP_STATUS" -ge 200 ] && [ "$HTTP_STATUS" -lt 500 ]; then
              echo "✅ API accessible ! (HTTP $HTTP_STATUS)"
              exit 0
            fi
            echo "⏳ Tentative $i/10..."
            sleep 3
          done
          echo "❌ L'API ne répond pas"
          exit 1

      - name: 🧹 Nettoyage
        if: always()
        run: |
          rm -f ~/.ssh/ec2_key
          rm -f ansible/inventory-ci.ini
```

> 💡 **Pourquoi un inventaire différent (`inventory-ci.ini`) ?** Parce que dans GitHub Actions, la clé SSH est dans `~/.ssh/ec2_key`, alors que sur votre machine elle est dans `~/.ssh/tp-api-keypair`.

## Étape 9.2 : Tester

```bash
git add ansible/ .github/workflows/deploy.yml
git commit -m "cd: déploiement via Ansible dans GitHub Actions"
git push origin main
```

Regardez dans **Actions** sur GitHub : Ansible s'exécute dans le workflow avec toute sa sortie colorée.

**❓ Question :** Quel avantage y a-t-il à utiliser Ansible dans GitHub Actions plutôt que des commandes SSH directes ?

---

# 🔄 PARTIE 10 : Commandes Ansible Rapides (Ad-Hoc)

Ansible n'est pas limité aux playbooks. Vous pouvez exécuter des **commandes rapides** sur vos serveurs :

```bash
cd ansible

# Voir l'espace disque
ansible webservers -a "df -h"

# Voir la mémoire
ansible webservers -a "free -m"

# Voir l'état de l'application
ansible webservers -a "pm2 status"

# Voir les derniers logs
ansible webservers -a "pm2 logs tp-api --lines 20 --nostream"

# Redémarrer l'application
ansible webservers -a "pm2 restart tp-api"

# Installer un outil de monitoring
ansible webservers -m yum -a "name=htop state=present" --become
```

> 💡 Les commandes ad-hoc sont utiles pour du diagnostic rapide. Pour des actions répétitives, utilisez un playbook.

**❓ Question :** Quand utiliser une commande ad-hoc plutôt qu'un playbook ?

---

# 🏗️ PARTIE 11 : Organiser avec les Rôles Ansible

## Le problème

Si vous avez plusieurs applications à déployer, copier-coller les mêmes tâches dans chaque playbook n'est pas pratique. Les **rôles** permettent de créer des "briques réutilisables".

## Étape 11.1 : Créer le rôle `nodejs-app`

```bash
cd ansible
mkdir -p roles/nodejs-app/{tasks,handlers,templates,defaults}
```

Créez `ansible/roles/nodejs-app/defaults/main.yml` :

```yaml
---
# Valeurs par défaut (modifiables dans le playbook)
app_user: "ec2-user"
app_dir: "/home/ec2-user/app"
app_port: 3000
node_env: "production"
node_version: "20"
github_repo: ""
github_branch: "main"
```

Créez `ansible/roles/nodejs-app/tasks/main.yml` :

```yaml
---
- name: 📥 Récupérer le code depuis GitHub
  git:
    repo: "{{ github_repo }}"
    dest: "{{ app_dir }}"
    version: "{{ github_branch }}"
    force: yes
  register: git_result
  when: github_repo != ""

- name: ⚙️ Générer le fichier .env
  template:
    src: env.j2
    dest: "{{ app_dir }}/.env"
    mode: '0600'
  notify: Redémarrer PM2

- name: 📦 Installer les dépendances
  npm:
    path: "{{ app_dir }}"
    production: yes
    state: present

- name: 🚀 Démarrer l'application
  command: pm2 startOrReload ecosystem.config.js --env production
  args:
    chdir: "{{ app_dir }}"

- name: 💾 Sauvegarder la config PM2
  command: pm2 save
  changed_when: false

- name: ⏳ Attendre le démarrage
  pause:
    seconds: 3

- name: 🏥 Vérifier que l'API répond
  uri:
    url: "http://localhost:{{ app_port }}/"
    method: GET
    status_code: [200, 301, 302, 404]
    timeout: 10
  retries: 5
  delay: 3
  register: health_check
  until: health_check.status != -1
```

Créez `ansible/roles/nodejs-app/handlers/main.yml` :

```yaml
---
- name: Redémarrer PM2
  command: pm2 reload ecosystem.config.js --env production
  args:
    chdir: "{{ app_dir }}"
```

Copiez le template :

```bash
cp ansible/templates/env.j2 ansible/roles/nodejs-app/templates/env.j2
```

## Étape 11.2 : Utiliser le rôle

Créez `ansible/playbook-deploy-role.yml` :

```yaml
---
- name: 🚀 Déploiement via le rôle nodejs-app
  hosts: webservers
  become: no

  roles:
    - role: nodejs-app
      vars:
        github_repo: "https://github.com/VOTRE_USERNAME/tp-api-nodejs.git"
        github_branch: "main"
        app_port: 3000
```

C'est beaucoup plus court ! Et si vous aviez 3 applications, il suffirait de réutiliser le rôle 3 fois avec des variables différentes.

```bash
ansible-playbook playbook-deploy-role.yml
```

**❓ Questions :**
1. Quel est l'avantage d'un rôle par rapport à tout mettre dans un seul fichier ?
2. Si vous aviez 3 applications Node.js, comment le rôle vous aiderait-il ?

---

# 🧹 PARTIE 12 : Nettoyage

Quand vous avez terminé le TP, **nettoyez tout** pour ne pas gaspiller vos crédits.

## Étape 12.1 : Arrêter l'application avec Ansible

Créez `ansible/playbook-cleanup.yml` :

```yaml
---
- name: 🧹 Nettoyage du serveur
  hosts: webservers
  become: no

  vars:
    app_dir: "/home/ec2-user/app"

  tasks:
    - name: 🛑 Arrêter l'application
      command: pm2 delete all
      ignore_errors: yes

    - name: 💾 Vider la liste PM2
      command: pm2 save --force
      ignore_errors: yes

    - name: 🗑️ Supprimer le code
      file:
        path: "{{ app_dir }}"
        state: absent
      become: yes

    - name: ✅ Nettoyage terminé
      debug:
        msg: "Le serveur a été nettoyé."
```

```bash
ansible-playbook playbook-cleanup.yml
```

## Étape 12.2 : Supprimer le serveur avec Terraform

```bash
cd terraform
terraform destroy
```

Terraform demande confirmation. Tapez `yes` :

```
Destroy complete! Resources: 3 destroyed. ✅
```

**Tout a été supprimé** : le serveur, le pare-feu, et la clé SSH dans AWS. En une seule commande.

> 💡 C'est l'un des grands avantages de Terraform : la suppression est aussi simple que la création.

## Étape 12.3 : Sauvegarder votre travail

```bash
cd ..
git add ansible/ terraform/
git commit -m "infra: configuration complète Terraform + Ansible"
git push origin main
```

---

# 📋 PARTIE 13 : Récapitulatif

## Structure finale du projet

```
tp-api-nodejs/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Tests automatiques
│       └── deploy.yml                # Déploiement via Ansible
├── terraform/
│   ├── main.tf                       # Serveur + pare-feu + clé SSH
│   ├── variables.tf                  # Paramètres
│   ├── outputs.tf                    # IP, URL, commande SSH
│   ├── terraform.tfvars              # Vos valeurs (non commité)
│   └── .gitignore
├── ansible/
│   ├── ansible.cfg                   # Configuration Ansible
│   ├── inventory.ini                 # Liste des serveurs
│   ├── playbook-setup.yml            # Installer Node.js, PM2, Git
│   ├── playbook-deploy.yml           # Déployer l'application
│   ├── playbook-deploy-role.yml      # Déployer via rôle
│   ├── playbook-cleanup.yml          # Nettoyer le serveur
│   ├── templates/
│   │   └── env.j2
│   └── roles/
│       └── nodejs-app/
│           ├── tasks/main.yml
│           ├── handlers/main.yml
│           ├── templates/env.j2
│           └── defaults/main.yml
├── ecosystem.config.js               # Configuration PM2
├── server.js
├── package.json
└── README.md
```

## Aide-mémoire Terraform

| Commande | Ce qu'elle fait |
|----------|----------------|
| `terraform init` | Préparer le projet |
| `terraform plan` | Voir ce qui va changer (sans rien faire) |
| `terraform apply` | Créer/modifier les ressources |
| `terraform destroy` | Tout supprimer |
| `terraform output` | Revoir les infos (IP, URL...) |
| `terraform validate` | Vérifier la syntaxe des fichiers |

## Aide-mémoire Ansible

| Commande | Ce qu'elle fait |
|----------|----------------|
| `ansible servers -m ping` | Tester la connexion |
| `ansible servers -a "commande"` | Exécuter une commande rapide |
| `ansible-playbook playbook.yml` | Lancer un playbook |
| `ansible-playbook playbook.yml --check` | Simuler sans exécuter |
| `ansible-playbook playbook.yml -e "var=val"` | Passer une variable |

## Qui fait quoi ? (schéma complet)

```
┌─────────────────────────────────────────────────┐
│              TERRAFORM                           │
│          (crée les machines)                     │
│                                                  │
│  terraform apply  → Serveur EC2 créé             │
│                   → Pare-feu configuré           │
│                   → Clé SSH enregistrée          │
│                                                  │
│  terraform destroy → Tout supprimé               │
└────────────────────────┬────────────────────────┘
                         │ IP du serveur
                         ▼
┌─────────────────────────────────────────────────┐
│              ANSIBLE                             │
│        (configure les machines)                  │
│                                                  │
│  playbook-setup   → Node.js, PM2, Git installés │
│  playbook-deploy  → Code déployé, app lancée    │
│  playbook-cleanup → Serveur nettoyé             │
└────────────────────────┬────────────────────────┘
                         │ intégré dans
                         ▼
┌─────────────────────────────────────────────────┐
│           GITHUB ACTIONS                         │
│      (automatise le tout)                        │
│                                                  │
│  push → CI (tests) → CD (Ansible) → App live !  │
└─────────────────────────────────────────────────┘
```

---

# 🏆 Défis

## Défi 1 : Rollback (⭐)

Créez un playbook `playbook-rollback.yml` qui revient à la version précédente si un déploiement casse quelque chose.

<details>
<summary>💡 Indice</summary>

```yaml
---
- name: ⏪ Rollback
  hosts: webservers
  become: no

  vars:
    app_dir: "/home/ec2-user/app"

  tasks:
    - name: 📋 Commit actuel
      command: git rev-parse HEAD
      args:
        chdir: "{{ app_dir }}"
      register: current_commit

    - name: ⏪ Revenir au commit précédent
      command: git checkout HEAD~1
      args:
        chdir: "{{ app_dir }}"

    - name: 📦 Réinstaller les dépendances
      npm:
        path: "{{ app_dir }}"
        production: yes
        state: present

    - name: 🔄 Redémarrer
      command: pm2 reload ecosystem.config.js --env production
      args:
        chdir: "{{ app_dir }}"

    - name: ✅ Rollback fait
      debug:
        msg: "Rollback depuis {{ current_commit.stdout[:8] }}"
```
</details>

---

## Défi 2 : Staging + Production (⭐⭐)

Créez deux environnements séparés : un serveur de staging et un serveur de production.

<details>
<summary>💡 Indice</summary>

1. Créez deux fichiers `terraform.tfvars` (un par environnement)
2. Créez deux inventaires Ansible

```bash
# Créer le staging
cd terraform
terraform apply -var-file=environments/staging.tfvars

# Créer la production
terraform workspace new production
terraform apply -var-file=environments/production.tfvars
```

```bash
# Déployer en staging
ansible-playbook -i inventories/staging.ini playbook-deploy.yml

# Déployer en production
ansible-playbook -i inventories/production.ini playbook-deploy.yml
```
</details>

---

## Défi 3 : Chiffrer les secrets avec Ansible Vault (⭐⭐⭐)

Utilisez Ansible Vault pour stocker des mots de passe et clés API de manière chiffrée.

<details>
<summary>💡 Indice</summary>

```bash
# Créer un fichier de secrets chiffré
ansible-vault create ansible/secrets.yml

# Contenu (dans l'éditeur qui s'ouvre) :
# mongodb_uri: "mongodb+srv://user:password@cluster.mongodb.net/tp-api"

# Lancer le playbook avec les secrets
ansible-playbook playbook-deploy.yml --ask-vault-pass
```

Dans le playbook :
```yaml
vars_files:
  - secrets.yml
```
</details>

---

# 🔧 Dépannage

## Terraform

| Problème | Solution |
|----------|----------|
| `terraform: command not found` | Installez Terraform (cf. étape 1.2) |
| `No valid credential sources found` | Ré-exportez les credentials Vocareum (cf. étape 1.6) |
| `UnauthorizedOperation` | Votre lab a expiré → relancez-le dans Vocareum |
| `already exists` | La ressource existe d'une session précédente → supprimez-la dans la console AWS |
| `terraform plan` montre des changements bizarres | Quelqu'un a modifié les ressources manuellement dans AWS |

## Ansible

| Problème | Solution |
|----------|----------|
| `ansible: command not found` | Installez Ansible (cf. étape 6.1) |
| `UNREACHABLE` | Vérifiez l'IP, la clé SSH, le port 22, l'état du serveur |
| `Permission denied (publickey)` | Vérifiez `chmod 400` sur la clé et le username `ec2-user` |
| `MODULE FAILURE` sur `yum` | Ajoutez `become: yes` (besoin de sudo) |
| PM2 affiche `errored` | Connectez-vous en SSH et regardez `pm2 logs` |
| Health check échoue | Vérifiez `pm2 logs` et le fichier `.env` |
| Credentials Vocareum expirées | Relancez le lab (Stop Lab → Start Lab) |

---

# 📊 Ce que vous avez accompli

- ✅ **Terraform** : créé un serveur, un pare-feu et une clé SSH avec du code
- ✅ **Configuration manuelle** : compris ce qu'il faut installer sur un serveur
- ✅ **GitHub Actions** : déploiement automatique à chaque push
- ✅ **Ansible** : automatisé l'installation des logiciels (playbook-setup)
- ✅ **Ansible** : automatisé le déploiement (playbook-deploy)
- ✅ **Rôles Ansible** : organisé le code pour le réutiliser
- ✅ **Pipeline complet** : push → tests → déploiement → API en ligne
- ✅ **Nettoyage** : tout supprimé proprement avec `terraform destroy`

```
TP 1 : API REST              →  Coder
TP 2 : Tests                 →  Vérifier
TP 3 : CI/CD GitHub Actions  →  Automatiser les tests
TP 4 : Terraform + Ansible   →  Créer le serveur et déployer automatiquement ← vous êtes ici !
```

---

# 🎉 Félicitations !

Votre API est déployée sur un vrai serveur dans le cloud, l'infrastructure est gérée par Terraform, la configuration est automatisée par Ansible, et le tout se déclenche automatiquement à chaque push grâce à GitHub Actions.

Vous avez les bases du métier **DevOps**. Bienvenue ! 🚀
