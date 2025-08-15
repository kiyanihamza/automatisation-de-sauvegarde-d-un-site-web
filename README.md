# Script de Backup Automatisé pour le site E-commerce

## 📋 Vue d'ensemble

Ce guide décrit la mise en place d'un script bash automatisé pour sauvegarder le site e-commerce depuis App Server 3 vers le serveur de backup Nautilus.

## 🎯 Objectif du projet

Créer un script `ecommerce_backup.sh` qui :
1. Archive le site `/var/www/html/ecommerce`
2. Sauvegarde localement dans `/backup`
3. Copie automatiquement sur le serveur de backup distant
4. Fonctionne sans intervention manuelle (SSH par clé)

## 🏗️ Architecture

```
App Server 3 (stapp03)
├─ /var/www/html/ecommerce → archive xfusioncorp_ecommerce.zip
└─ /backup/xfusioncorp_ecommerce.zip (local)

          scp (SSH key-based)
                    ↓
    
Nautilus Backup Server (stbkp01)
└─ /backup/xfusioncorp_ecommerce.zip (distant)
```

## 📋 Prérequis

| Élément | Détail |
|---------|--------|
| **Serveur source** | App Server 3 (`stapp03`) |
| **Serveur backup** | Nautilus Backup Server (`stbkp01`) |
| **Utilisateur local** | `banner` (ou utilisateur courant) |
| **Utilisateur distant** | `clint` |
| **Répertoires requis** | `/backup` sur les deux serveurs |

## 🔐 Configuration SSH par clé

### Étape 1 : Génération de la paire de clés

```bash
ssh-keygen -t rsa -b 4096 -C "banner@stapp03"
```

- **Clé privée** : `~/.ssh/id_rsa` (reste sur App Server 3)
- **Clé publique** : `~/.ssh/id_rsa.pub` (à copier sur le serveur backup)

### Étape 2 : Copie de la clé publique

```bash
ssh-copy-id clint@stbkp01
```

### Étape 3 : Vérification de la connexion

```bash
ssh clint@stbkp01
```

✅ La connexion doit s'établir **sans demande de mot de passe**

## 📜 Script de backup

### Emplacement

```
/scripts/ecommerce_backup.sh
```

### Contenu complet

```bash
#!/bin/bash

# ========================================
# Script de backup automatisé E-commerce
# ========================================

# Variables de configuration
SRC_DIR="/var/www/html/ecommerce"
LOCAL_BACKUP_DIR="/backup"
REMOTE_USER="clint"
REMOTE_HOST="stbkp01"
REMOTE_BACKUP_DIR="/backup"
ARCHIVE_NAME="xfusioncorp_ecommerce.zip"

# Couleurs pour les messages
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

echo "🚀 Démarrage du backup E-commerce..."

# 1. Vérification des prérequis
if [ ! -d "$SRC_DIR" ]; then
    echo -e "${RED}❌ Erreur: Répertoire source $SRC_DIR introuvable${NC}"
    exit 1
fi

if [ ! -d "$LOCAL_BACKUP_DIR" ]; then
    echo -e "${RED}❌ Erreur: Répertoire backup local $LOCAL_BACKUP_DIR introuvable${NC}"
    exit 1
fi

# 2. Création de l'archive (mode silencieux)
echo "📦 Création de l'archive..."
zip -r -q "/tmp/$ARCHIVE_NAME" "$SRC_DIR"

if [ $? -ne 0 ]; then
    echo -e "${RED}❌ Erreur lors de la création de l'archive${NC}"
    exit 1
fi

# 3. Copie en local dans /backup
echo "💾 Copie locale..."
mv "/tmp/$ARCHIVE_NAME" "$LOCAL_BACKUP_DIR/"

if [ $? -ne 0 ]; then
    echo -e "${RED}❌ Erreur lors de la copie locale${NC}"
    exit 1
fi

# 4. Envoi sur le serveur distant via SCP
echo "🌐 Envoi vers le serveur backup..."
scp "$LOCAL_BACKUP_DIR/$ARCHIVE_NAME" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_BACKUP_DIR/"

if [ $? -ne 0 ]; then
    echo -e "${RED}❌ Erreur lors de l'envoi distant${NC}"
    exit 1
fi

# 5. Confirmation finale
echo -e "${GREEN}✅ Backup terminé avec succès : $ARCHIVE_NAME${NC}"
echo "📍 Local: $LOCAL_BACKUP_DIR/$ARCHIVE_NAME"
echo "📍 Distant: $REMOTE_USER@$REMOTE_HOST:$REMOTE_BACKUP_DIR/$ARCHIVE_NAME"
```

### Permissions du script

```bash
# Option sécurisée (recommandée)
chmod 700 /scripts/ecommerce_backup.sh

# Ou option standard
chmod 755 /scripts/ecommerce_backup.sh
```

## 🚀 Utilisation

### Exécution manuelle

```bash
# Méthode 1
bash /scripts/ecommerce_backup.sh

# Méthode 2 (si chmod +x)
/scripts/ecommerce_backup.sh
```

### Vérification des résultats

```bash
# Vérification locale
ls -l /backup/xfusioncorp_ecommerce.zip

# Vérification distante
ssh clint@stbkp01 "ls -l /backup/xfusioncorp_ecommerce.zip"
```

## ⏰ Automatisation (optionnelle)

### Ajout dans crontab

```bash
# Éditer la crontab
crontab -e

# Backup quotidien à 2h du matin
0 2 * * * /scripts/ecommerce_backup.sh >/dev/null 2>&1

# Backup hebdomadaire le dimanche à 3h
0 3 * * 0 /scripts/ecommerce_backup.sh >/dev/null 2>&1
```

## 🗂️ Gestion des anciennes sauvegardes

### Version avec horodatage

Pour conserver un historique, modifier la variable :

```bash
ARCHIVE_NAME="xfusioncorp_ecommerce_$(date +%F_%H-%M-%S).zip"
```

### Nettoyage automatique

Ajouter ces lignes dans le script pour supprimer les backups de plus de 7 jours :

```bash
# Nettoyage local (fichiers > 7 jours)
find "$LOCAL_BACKUP_DIR" -name "xfusioncorp_ecommerce*.zip" -type f -mtime +7 -exec rm -f {} \;

# Nettoyage distant (fichiers > 7 jours)
ssh "$REMOTE_USER@$REMOTE_HOST" "find $REMOTE_BACKUP_DIR -name 'xfusioncorp_ecommerce*.zip' -type f -mtime +7 -exec rm -f {} \;"
```

## 🔍 Troubleshooting

### Problèmes courants

| Problème | Solution |
|----------|----------|
| **Permission denied** | Vérifier `chmod 700` ou `755` sur le script |
| **SSH demande un mot de passe** | Reconfigurer les clés SSH avec `ssh-copy-id` |
| **Répertoire /backup inexistant** | Créer avec `sudo mkdir -p /backup` |
| **Archive corrompue** | Vérifier l'espace disque disponible |
| **SCP échoue** | Tester manuellement : `scp test.txt clint@stbkp01:/backup/` |

### Commandes de diagnostic

```bash
# Test connexion SSH
ssh -v clint@stbkp01

# Vérification des permissions
ls -la /scripts/ecommerce_backup.sh

# Test manuel de création d'archive
zip -r test.zip /var/www/html/ecommerce

# Vérification de l'espace disque
df -h /backup
```

## ✅ Checklist de déploiement

- [ ] Clés SSH configurées et testées
- [ ] Répertoires `/backup` créés sur les deux serveurs
- [ ] Script placé dans `/scripts/ecommerce_backup.sh`
- [ ] Permissions correctes appliquées (`chmod 700`)
- [ ] Test d'exécution manuelle réussi
- [ ] Vérification des archives créées (local + distant)
- [ ] (Optionnel) Automatisation crontab configurée

## 📚 Notes importantes

⚠️ **Sécurité** : 
- Les clés privées SSH ne doivent jamais quitter le serveur source
- Utiliser `chmod 700` pour restreindre l'accès au script

⚡ **Performance** : 
- L'archive est créée en mode silencieux (`-q`) pour éviter les logs verbeux
- Le fichier temporaire est créé dans `/tmp` puis déplacé

🔄 **Maintenance** : 
- Surveiller l'espace disque des répertoires de backup
- Mettre en place une rotation des sauvegardes si nécessaire

---

*Dernière mise à jour : [Date]*  
*Auteur : [Votre nom]*  
*Version : 1.0*
