# RAPPORT DE RÉALISATION

## Mise en Œuvre d'une Infrastructure Cloud de Supervision Centralisée sous AWS
### Déploiement de Zabbix Conteneurisé pour le Monitoring d'un Parc Hybride (Linux & Windows)

---

## TABLE DES MATIÈRES

1. Introduction
2. Architecture Proposée
3. Ressources AWS Déployées
4. Installation et Configuration
5. Challenges Rencontrés et Solutions
6. Résultats Finaux
7. Conclusion

---

## 1. INTRODUCTION

### Objectif du Projet
Déployer une infrastructure de monitoring centralisée sur AWS utilisant Zabbix (conteneurisé avec Docker) pour surveiller un parc hybride composé de serveurs Linux et Windows.

### Contexte
Ce projet a été réalisé dans le contexte du AWS Learner Lab avec des ressources limitées. L'infrastructure doit permettre:
- ✅ Supervision centralisée depuis une interface web
- ✅ Monitoring d'agents Linux et Windows
- ✅ Gestion des alertes et graphiques de performance
- ✅ Respect des contraintes de sécurité AWS

---

## 2. ARCHITECTURE PROPOSÉE

### 2.1 Architecture Réseau

**VPC Configuration:**
- **VPC**: 1 VPC avec sous-réseau public (172.20.0.0/16 pour Docker)
- **Subnet interne AWS**: 10.0.0.0/16
- **Nombre d'instances**: 3 instances EC2 (Zabbix Server + 2 Clients)

### 2.2 Composants du Système

```
┌─────────────────────────────────────────┐
│         Zabbix Server                   │
│  (Ubuntu 22.04 + Docker)                │
│  IP Privée: 10.0.1.166                  │
│                                         │
│  ├─ MySQL Database                      │
│  ├─ Zabbix Server Container             │
│  ├─ Zabbix Web Interface (Nginx)        │
│  └─ Zabbix Agent (localhost)            │
└─────────────────────────────────────────┘
         ↓                    ↓
    TCP 10050/10051      TCP 10050/10051
         ↓                    ↓
┌──────────────────┐  ┌──────────────────┐
│  Linux Client    │  │  Windows Client  │
│  Ubuntu 22.04    │  │  Windows Server  │
│  IP: 10.0.1.188  │  │  IP: 10.0.1.175  │
│                  │  │                  │
│ Zabbix Agent 2   │  │ Zabbix Agent     │
└──────────────────┘  └──────────────────┘
```

### 2.3 Spécifications des Instances

| Composant | Type Instance | OS | RAM | CPU | Rôle |
|-----------|---------------|----|----|-----|------|
| Zabbix Server | t3.medium | Ubuntu 22.04 | 8GB | 2vCPU | Serveur central + DB |
| Linux Client | t3.medium | Ubuntu 22.04 | 4GB | 2vCPU | Agent de monitoring |
| Windows Client | t3.medium | Windows Server | 4GB | 2vCPU | Agent de monitoring |

---

## 3. RESSOURCES AWS DÉPLOYÉES

### 3.1 Instances EC2

**Zabbix-Server:**
- IP Publique: 13.221.136.233 (change après redémarrage lab)
- IP Privée: 10.0.1.166
- Rôle: Serveur central de monitoring
- Services: Docker, MySQL, Zabbix Server, Zabbix Web

**Aya-Linux-Client:**
- IP Publique: 3.235.42.215
- IP Privée: 10.0.1.188
- Rôle: Client Linux à surveiller
- Services: Zabbix Agent 2

**Aya-Windows-Client:**
- IP Publique: 98.92.60.12
- IP Privée: 10.0.1.175
- Rôle: Client Windows à surveiller
- Services: Zabbix Agent

### 3.2 Sécurité Réseau (Security Groups)

**Règles Inbound pour Zabbix Server:**
```
Port 10051 (TCP) → Source: 10.0.1.188/32, 10.0.1.175/32
Port 80 (HTTP) → Source: 0.0.0.0/0
Port 443 (HTTPS) → Source: 0.0.0.0/0
Port 22 (SSH) → Source: Votre IP
Port 3306 (MySQL) → Source: Conteneurs Docker
```

**Règles Inbound pour Clients:**
```
Port 10050 (TCP) → Source: 10.0.1.166/32 (Zabbix Server)
Port 22 (SSH Linux) → Source: Votre IP
Port 3389 (RDP Windows) → Source: Votre IP
```

---

## 4. INSTALLATION ET CONFIGURATION

### 4.1 Déploiement du Serveur Zabbix

#### Étape 1: Préparation de l'instance Ubuntu

```bash
ssh -i aya_key.pem ubuntu@13.221.136.233

# Mise à jour du système
sudo apt update
sudo apt upgrade -y

# Installation de Docker
sudo apt install docker.io docker-compose -y
sudo usermod -aG docker ubuntu
```

#### Étape 2: Configuration Docker-Compose

Fichier `docker-compose.yml` déployé:

```yaml
version: '3.8'
services:
  zabbix-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - zabbix-db-data:/var/lib/mysql
    ports:
      - "3306:3306"

  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-6.4-latest
    ports:
      - "10051:10051"
    environment:
      DB_SERVER_HOST: zabbix-db
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix
    depends_on:
      - zabbix-db

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:ubuntu-6.4-latest
    ports:
      - "80:8080"
      - "443:8443"
    environment:
      DB_SERVER_HOST: zabbix-db
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: "Europe/Paris"

  zabbix-agent:
    image: zabbix/zabbix-agent:ubuntu-6.4-latest
    ports:
      - "10050:10050"
    environment:
      ZBX_SERVER_HOST: zabbix-server
      ZBX_HOSTNAME: Zabbix-server
```

#### Étape 3: Démarrage des Conteneurs

```bash
cd ~/zabbix-aya-docker
docker-compose up -d
docker ps  # Vérification des conteneurs actifs
```

**Résultat:**
```
✅ zabbix-db → Status: healthy
✅ zabbix-server → Status: running (Port 10051 écoute)
✅ zabbix-web → Status: healthy (Port 80/443)
✅ zabbix-agent → Status: running (Port 10050)
```

### 4.2 Configuration du Client Linux

#### Étape 1: Installation de l'Agent Zabbix

```bash
ssh -i aya_key.pem ubuntu@3.235.42.215

# Téléchargement du package Zabbix
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo apt update

# Installation de l'agent
sudo apt install -y zabbix-agent
```

#### Étape 2: Configuration de l'Agent

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

**Paramètres modifiés:**
```ini
Server=10.0.1.166              # IP privée du serveur Zabbix
ServerActive=10.0.1.166:10051  # Pour les checks actifs
Hostname=Aya-Linux-Client      # Nom unique de l'agent
ListenPort=10050               # Port d'écoute
ListenIP=0.0.0.0               # Écoute sur toutes les interfaces
```

#### Étape 3: Démarrage du Service

```bash
sudo systemctl start zabbix-agent
sudo systemctl enable zabbix-agent
sudo systemctl status zabbix-agent

# Vérification du port
ss -tlnp | grep 10050
```

**Résultat:**
```
✅ LISTEN 0.0.0.0:10050 (Agent écoute correctement)
```

### 4.3 Configuration du Client Windows

#### Étape 1: Installation de l'Agent Windows

**Étapes manuelles:**
1. Télécharger l'agent Zabbix 6.4 pour Windows depuis zabbix.com
2. Extraire vers `C:\Program Files\Zabbix Agent`
3. Ouvrir PowerShell en tant qu'Administrateur

#### Étape 2: Configuration du Fichier Config

```powershell
notepad "C:\Program Files\Zabbix Agent\zabbix_agentd.conf"
```

**Paramètres modifiés:**
```ini
Server=10.0.1.166              # IP privée du serveur Zabbix
ServerActive=10.0.1.166:10051  # Pour les checks actifs
Hostname=Aya-Windows-Client    # Nom unique de l'agent
ListenPort=10050               # Port d'écoute
ListenIP=0.0.0.0               # Écoute sur toutes les interfaces
```

#### Étape 3: Installation en tant que Service

```powershell
cd "C:\Program Files\Zabbix Agent"
.\zabbix_agentd.exe -i -c zabbix_agentd.conf

# Démarrage du service
Start-Service "Zabbix Agent"
Get-Service "Zabbix Agent"  # Vérification

# Vérification du port
netstat -ano | findstr 10050
```

**Résultat:**
```
✅ Status: Running
✅ Port 10050: LISTENING
```

#### Étape 4: Configuration du Firewall Windows

```powershell
# Ouvrir Windows Defender Firewall with Advanced Security (wf.msc)
# Ajouter une règle Inbound:
# - Port: 10050
# - Protocole: TCP
# - Action: Allow
# - Profiles: Domain, Private, Public
```

### 4.4 Création des Hosts dans Zabbix Web

#### Accès à l'Interface

```
URL: http://13.221.136.233
Utilisateur par défaut: admin
Mot de passe par défaut: zabbix
```

#### Création Host 1: Zabbix-server

```
Configuration → Hosts → Create Host

Host name: Zabbix-server
Groups: Zabbix servers (créer le groupe)
Interface Agent:
  - IP: 127.0.0.1
  - Port: 10050
Status: Enabled ✅
```

#### Création Host 2: Aya-Linux-Client

```
Host name: Aya-Linux-Client
Groups: Linux servers
Interface Agent:
  - IP: 10.0.1.188
  - Port: 10050
Status: Enabled ✅
```

#### Création Host 3: Aya-Windows-Client

```
Host name: Aya-Windows-Client
Groups: Windows servers
Interface Agent:
  - IP: 10.0.1.175
  - Port: 10050
Status: Enabled ✅
```

---

## 5. CHALLENGES RENCONTRÉS ET SOLUTIONS

### Challenge 1: Port 10051 Non Accessible Initialement

**Problème:**
```
Container zabbix-server créé mais port 10051 n'écoute pas
Erreur: "Unable to connect to [zabbix-server]:10051: Connection refused"
```

**Cause:**
- Serveur Zabbix n'était pas complètement initialisé
- La base de données MySQL n'était pas prête

**Solution:**
```bash
# Attendre 5-10 minutes pour l'initialisation complète
sleep 300
docker logs zabbix-server | grep "server #0 started"
```

---

### Challenge 2: Agent Linux: zabbix-agent vs zabbix-agent2

**Problème:**
```
Agent2 était installé, mais Zabbix cherche l'agent standard
Les deux utilisent des protocoles différents
```

**Cause:**
- La machine avait zabbix-agent2 (protocole v2 incompatible)
- Zabbix 6.4 attendait zabbix-agent (protocole standard)

**Solution:**
```bash
# Désinstaller agent2
sudo systemctl stop zabbix-agent2
sudo systemctl disable zabbix-agent2

# Installer l'agent standard
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo apt install -y zabbix-agent
```

---

### Challenge 3: Firewall Windows Bloquant le Port 10050

**Problème:**
```
Agent Windows tourne mais le serveur Zabbix ne peut pas le contacter
Erreur réseau "Connection refused"
```

**Cause:**
- Windows Firewall bloquait les connexions entrantes sur port 10050
- Règle firewall non configurée

**Solution:**
```powershell
# Créer une règle firewall
New-NetFirewallRule -DisplayName "Zabbix Agent" `
  -Direction Inbound -Protocol TCP -LocalPort 10050 `
  -Action Allow -Profile Any
```

---

### Challenge 4: Security Groups AWS Trop Restrictifs

**Problème:**
```
Les agents ne pouvaient pas envoyer les données au serveur Zabbix
Port 10051 bloqué entre les instances
```

**Cause:**
- Security Group du serveur Zabbix n'autorisait pas l'inbound sur port 10051
- Source IP des agents non whitelistées

**Solution:**
```
AWS Console → Security Groups → Zabbix Server SG
Ajouter règle Inbound:
- Port: 10051
- Protocole: TCP
- Source: 10.0.1.188/32 (Linux)
          10.0.1.175/32 (Windows)
```

---

### Challenge 5: Redémarrage du Lab → Changement d'IPs Publiques

**Problème:**
```
Après redémarrage AWS Learner Lab, les IPs publiques ont changé
Les anciennes IPs ne fonctionnaient plus
```

**Cause:**
- AWS Learner Lab réattribue les IPs publiques après un arrêt

**Solution:**
```
Mise à jour UNIQUEMENT des IPs publiques:
- Ne PAS modifier les IPs privées
- Ne PAS modifier les configs des agents (utilisent IPs privées)
- Relancer Docker et les agents avec:

docker-compose up -d
sudo systemctl restart zabbix-agent
```

---

### Challenge 6: Pas d'Agent Installé sur Linux Client

**Problème:**
```
Hosts "Unknown" dans Zabbix Web
Logs: "host [Aya-Linux-Client] not found"
```

**Cause:**
- After lab restart, zabbix-agent n'était pas relancé
- Service n'était pas actif (running)

**Solution:**
```bash
sudo systemctl start zabbix-agent
sudo systemctl enable zabbix-agent
```

---

## 6. RÉSULTATS FINAUX

### 6.1 État des Hosts Zabbix

**Après la configuration complète:**

| Host | Status | Interface | Données |
|------|--------|-----------|---------|
| Zabbix-server | 🟢 Available | 127.0.0.1:10050 | ✅ OK |
| Aya-Linux-Client | 🟢 Available | 10.0.1.188:10050 | ✅ OK |
| Aya-Windows-Client | 🟢 Available | 10.0.1.175:10050 | ✅ OK |

### 6.2 Métriques Collectées

**Linux Client:**
- CPU utilization
- Memory usage
- Disk I/O
- Network traffic
- System uptime

**Windows Client:**
- CPU utilization
- Memory usage
- Disk space
- Process count
- Network interfaces

**Zabbix Server:**
- Server internal metrics
- Database connectivity
- Trapper performance

### 6.3 Connectivité Réseau Vérifiée

```bash
# Tests effectués avec succès:
nc -zv 10.0.1.188 10050  → ✅ succeeded
nc -zv 10.0.1.175 10050  → ✅ succeeded

# Ports écoutant:
docker exec zabbix-server ss -tlnp | grep 10051 → ✅ 0.0.0.0:10051
ssh ubuntu@10.0.1.188 'ss -tlnp | grep 10050' → ✅ 0.0.0.0:10050
netstat -ano | findstr 10050 (Windows) → ✅ LISTENING
```

---

## 7. CONCLUSION

### Bilan du Projet

✅ **Objectifs Atteints:**
1. ✅ Infrastructure Zabbix 6.4 déployée sur AWS
2. ✅ 2 Clients (Linux + Windows) intégrés et monitoring actif
3. ✅ Interface Web fonctionnelle et accessible
4. ✅ Collecte de données en temps réel
5. ✅ Sécurité réseau configurée (Security Groups + Firewalls)

### Compétences Acquises

- **DevOps:** Docker, Docker-Compose
- **Cloud:** AWS EC2, Security Groups, VPC
- **Monitoring:** Zabbix configuration et agent deployment
- **Linux:** Configuration système, systemctl, firewall
- **Windows:** Service configuration, Windows Firewall
- **Networking:** TCP/IP, ports, firewall rules

### Leçons Apprises

1. **Timeouts et Initialization:** Toujours attendre que les conteneurs Docker se stabilisent (5-10 min)
2. **Agent Compatibility:** Vérifier la version de l'agent correspond à la version Zabbix
3. **Security Groups:** Whitelister les IPs privées pour la communication interne
4. **Firewall Windows:** Créer les règles firewall avant de configurer les services
5. **Lab Restarts:** Les IPs publiques changent, pas les IPs privées

### Améliorations Futures

- [ ] Ajouter des templates Zabbix pour plus de métriques
- [ ] Configurer des alertes (email, Slack)
- [ ] Implémenter un proxy Zabbix pour la scalabilité
- [ ] Sauvegarder les configs dans Git
- [ ] Automatiser avec Terraform/CloudFormation
- [ ] Ajouter monitoring SNMP pour les équipements réseau

---

## ANNEXE: COMMANDES UTILES

### Vérification de l'État

```bash
# Zabbix Server
docker ps
docker logs zabbix-server
docker exec zabbix-server ss -tlnp

# Linux Agent
sudo systemctl status zabbix-agent
sudo ss -tlnp | grep 10050
sudo systemctl restart zabbix-agent

# Windows Agent (PowerShell Admin)
Get-Service "Zabbix Agent"
netstat -ano | findstr 10050
Restart-Service "Zabbix Agent"

# Connectivité
nc -zv 10.0.1.188 10050
nc -zv 10.0.1.175 10050
```

### Configuration Zabbix Web

```
URL: http://<PUBLIC_IP_ZABBIX_SERVER>
Login: admin
Password: zabbix
```

---

**Date de Réalisation:** Janvier 2026  
**Étudiant:** Aya  
**Encadrant:** Prof. Azeddine KHIAT  
**Filière:** Cloud & DevOps
