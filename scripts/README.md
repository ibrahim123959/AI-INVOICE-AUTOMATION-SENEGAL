# Scripts - Utilitaires Administration et Maintenance

## 📖 Contexte

Ce dossier contient des scripts autonomes pour opérations d'administration, maintenance et déploiement. Contrairement au code applicatif (backend/frontend), ces scripts sont exécutés ponctuellement par les administrateurs système, pas par les utilisateurs finaux.

**Types d'opérations :**
- Initialisation environnement (setup base de données)
- Maintenance récurrente (backups, nettoyage)
- Déploiement (mise en production)
- Génération données test (développement)
```
┌─────────────────────────────────────────┐
│  Administrateur Système                 │
│  (DevOps, DBA, Tech Lead)               │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    ► SCRIPTS ◄ (Ce module)               │
│                                          │
│  One-shot operations :                   │
│  - Setup BDD                             │
│  - Backups                               │
│  - Déploiement                           │
│  - Génération test data                  │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  Infrastructure (BDD, serveurs, etc.)    │
└──────────────────────────────────────────┘
```

## 📜 Scripts Disponibles

### 🗄️ setup_database.py

**Objectif :** Initialisation complète base de données

**Ce qu'il fait :**
1. Crée toutes les tables (invoices, extractions, imputations, etc.)
2. Applique migrations Alembic (schéma à jour)
3. Crée indexes pour performances
4. Vérifie intégrité référentielle
5. Affiche résumé (tables créées, contraintes, indexes)

**Usage :**
```bash
# Initialisation standard
python scripts/setup_database.py

# Avec verbose (détails SQL)
python scripts/setup_database.py --verbose

# Avec confirmation interactive
python scripts/setup_database.py --interactive
```

**Output exemple :**
```
🗄️  Database Setup - Saisie Auto Factures
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Database URL: postgresql://localhost/saisie_auto
Checking connection... ✓

Creating tables...
  ✓ invoices (9 columns)
  ✓ extractions (12 columns)
  ✓ imputations (8 columns)
  ✓ account_rules (6 columns)
  ✓ chart_of_accounts (5 columns)
  ✓ learning_events (7 columns)
  ✓ sage_exports (5 columns)

Creating indexes...
  ✓ idx_invoices_status
  ✓ idx_extractions_invoice_id
  ✓ idx_imputations_invoice_id

Applying migrations...
  ✓ All migrations up to date (revision: a3f7b2c9)

✅ Database setup complete!
   7 tables created
   3 indexes created
   Ready for data seeding
```

**Quand utiliser :**
- Nouveau déploiement (dev, staging, production)
- Après changements schéma BDD
- Reset environnement test

---

### 📊 load_chart_of_accounts.py

**Objectif :** Charger plan comptable SYSCOHADA dans BDD

**Ce qu'il fait :**
1. Lit fichier `backend/app/database/seeds/syscohada_chart.sql`
2. Parse comptes (code, libellé, type, parent)
3. Insère dans table `chart_of_accounts`
4. Gère doublons (skip si existe déjà)
5. Affiche statistiques (comptes insérés par type)

**Usage :**
```bash
# Chargement standard
python scripts/load_chart_of_accounts.py

# Avec fichier custom
python scripts/load_chart_of_accounts.py --file custom_chart.sql

# Mode replace (supprime existant avant)
python scripts/load_chart_of_accounts.py --replace
```

**Output exemple :**
```
📊 Loading Chart of Accounts - SYSCOHADA
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Source file: backend/app/database/seeds/syscohada_chart.sql
Parsing accounts...
  Found 487 accounts

Inserting into database...
  ✓ Classe 1 - Comptes de ressources durables (78 accounts)
  ✓ Classe 2 - Comptes d'actif immobilisé (92 accounts)
  ✓ Classe 3 - Comptes de stocks (41 accounts)
  ✓ Classe 4 - Comptes de tiers (89 accounts)
  ✓ Classe 5 - Comptes de trésorerie (34 accounts)
  ✓ Classe 6 - Comptes de charges (98 accounts)
  ✓ Classe 7 - Comptes de produits (55 accounts)

✅ Chart of accounts loaded successfully!
   487 accounts inserted
   0 duplicates skipped
```

**Quand utiliser :**
- Après `setup_database.py` (première fois)
- Mise à jour plan comptable (nouveaux comptes)
- Reset plan comptable (avec --replace)

---

### 💾 backup_database.py

**Objectif :** Sauvegarde complète base de données

**Ce qu'il fait :**
1. Dump PostgreSQL complet (schema + data)
2. Compresse en .gz (économie espace)
3. Nomme avec timestamp (traçabilité)
4. Sauvegarde dans dossier `backups/`
5. Optionnel : Upload vers cloud (S3/GCS)
6. Rotation automatique (garde 30 derniers jours)

**Usage :**
```bash
# Backup standard
python scripts/backup_database.py

# Backup avec upload S3
python scripts/backup_database.py --upload-s3

# Backup sans compression
python scripts/backup_database.py --no-compress

# Dry-run (simule sans exécuter)
python scripts/backup_database.py --dry-run
```

**Output exemple :**
```
💾 Database Backup - Saisie Auto Factures
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Database: postgresql://localhost/saisie_auto
Timestamp: 2024-11-15_14-30-45

Creating backup...
  ✓ Schema dumped (2.3 MB)
  ✓ Data dumped (45.7 MB)
  ✓ Compressed to 8.2 MB (82% reduction)

Saving backup...
  ✓ Saved to: backups/backup_20241115_143045.sql.gz

Rotating old backups...
  ✓ Keeping 30 most recent backups
  ✓ Deleted 2 old backups (60+ days)

✅ Backup complete!
   Size: 8.2 MB
   Location: backups/backup_20241115_143045.sql.gz
   
Restore command:
  gunzip -c backups/backup_20241115_143045.sql.gz | psql saisie_auto
```

**Configuration cron (automatisation) :**
```bash
# Backup quotidien à 2h du matin
0 2 * * * cd /path/to/project && python scripts/backup_database.py --upload-s3
```

**Quand utiliser :**
- Quotidiennement (cron job production)
- Avant migrations BDD risquées
- Avant mises à jour majeures
- Avant opérations destructrices

---

### 🧪 generate_test_data.py

**Objectif :** Générer données factices pour tests et démos

**Ce qu'il fait :**
1. Crée factures fictives réalistes (fournisseurs sénégalais courants)
2. Génère extractions simulées (avec variations réalistes)
3. Crée imputations suggérées (avec scores confiance variés)
4. Insère données cohérentes en BDD
5. Respecte contraintes (dates cohérentes, montants positifs, etc.)

**Usage :**
```bash
# Génère 50 factures (défaut)
python scripts/generate_test_data.py

# Génère 200 factures
python scripts/generate_test_data.py --count 200

# Période spécifique
python scripts/generate_test_data.py --start-date 2024-01-01 --end-date 2024-03-31

# Fournisseurs spécifiques
python scripts/generate_test_data.py --suppliers SENELEC,ORANGE,WAVE

# Avec anomalies (pour tester détection)
python scripts/generate_test_data.py --include-anomalies
```

**Output exemple :**
```
🧪 Generating Test Data
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration:
  Count: 50 invoices
  Period: 2024-01-01 to 2024-11-15
  Suppliers: SENELEC, ORANGE, WAVE, SONATEL, CDE

Generating invoices...
  ✓ 12 SENELEC invoices (electricity)
  ✓ 8 ORANGE invoices (mobile money)
  ✓ 6 WAVE invoices (mobile money)
  ✓ 10 SONATEL invoices (telecom)
  ✓ 14 CDE invoices (water)

Generating extractions...
  ✓ 50 extractions created
  ✓ Average confidence: 0.89
  ✓ 3 low confidence (<0.70) for testing

Generating imputations...
  ✓ 50 imputations created
  ✓ 42 high confidence (>0.85)
  ✓ 8 medium confidence (0.70-0.85)

✅ Test data generated successfully!
   50 invoices created
   Ready for testing frontend/validation workflows
```

**Fournisseurs générés (réalistes Sénégal) :**
- SENELEC (électricité) → Compte 6054
- ORANGE (mobile money) → Compte 6241
- WAVE (mobile money) → Compte 6241
- SONATEL (téléphone) → Compte 6261
- CDE (eau) → Compte 6055
- Locations diverses → Compte 6132
- Fournitures bureau → Compte 6064

**Quand utiliser :**
- Après setup BDD (environnement dev)
- Préparation démos clients
- Tests frontend (besoin factures pour UI)
- Tests performance (volumes élevés)

---

### 🚀 deploy.sh

**Objectif :** Script déploiement automatisé production

**Ce qu'il fait :**
1. Vérifie pré-requis (Git propre, tests passent)
2. Pull dernières images Docker
3. Applique migrations BDD (zero-downtime)
4. Restart services (backend, ml-engine, frontend)
5. Health checks (vérifie services démarrés)
6. Rollback automatique si échec
7. Notifications Slack (succès/échec)

**Usage :**
```bash
# Déploiement production
./scripts/deploy.sh production

# Déploiement staging
./scripts/deploy.sh staging

# Dry-run (simule sans exécuter)
./scripts/deploy.sh production --dry-run

# Force (skip confirmations)
./scripts/deploy.sh production --force
```

**Output exemple :**
```
🚀 Deployment Script - Saisie Auto Factures
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Environment: production
Branch: main
Commit: a3f7b2c - feat: add duplicate detection (John Doe)

Pre-flight checks...
  ✓ Git working directory clean
  ✓ On main branch
  ✓ All tests passing
  ✓ Docker daemon running

Creating backup...
  ✓ Database backup created: backup_20241115_143045.sql.gz

Pulling latest images...
  ✓ backend:latest (digest: sha256:a3f7...)
  ✓ ml-engine:latest (digest: sha256:b2e9...)
  ✓ frontend:latest (digest: sha256:c4d1...)

Applying database migrations...
  ✓ Migration 001_add_learning_events applied
  ✓ All migrations up to date

Restarting services...
  ✓ backend restarted (3 replicas)
  ✓ ml-engine restarted (2 replicas)
  ✓ frontend restarted (2 replicas)

Health checks...
  ✓ Backend responding (200 OK)
  ✓ ML Engine responding (200 OK)
  ✓ Frontend serving (200 OK)

Notifying team...
  ✓ Slack notification sent to #deployments

✅ Deployment successful!
   Version: v1.2.3
   Duration: 3m 42s
   Deployed by: john.doe@example.com
   
Rollback command (if needed):
  ./scripts/deploy.sh production --rollback v1.2.2
```

**Rollback automatique :**

Si health checks échouent après déploiement, rollback automatique vers version précédente :
```
❌ Health check failed: Backend not responding

Rolling back...
  ✓ Reverting to previous images (v1.2.2)
  ✓ Services restarted
  ✓ Health checks passed

⚠️  Deployment rolled back due to health check failure
    Check logs: docker-compose logs backend
```

**Quand utiliser :**
- Mise en production features validées
- Déploiement fixes critiques
- Mise à jour versions (weekly release)

---

## 🔧 Configuration

### Variables Environnement

Tous scripts lisent variables depuis `.env` ou environnement système :
```bash
# .env
DATABASE_URL=postgresql://user:pass@localhost:5432/saisie_auto
BACKUP_PATH=/var/backups/saisie-auto
S3_BUCKET=saisie-auto-backups
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

### Permissions
```bash
# Scripts exécutables
chmod +x scripts/*.sh
chmod +x scripts/deploy.sh

# Scripts Python (pas besoin +x, appelés via python)
```

## 🧪 Tests Scripts

### Dry-Run Mode

Tous scripts supportent `--dry-run` pour tester sans modifier données :
```bash
python scripts/backup_database.py --dry-run
# Output : Simule backup sans créer fichier

python scripts/setup_database.py --dry-run
# Output : Affiche SQL qui serait exécuté sans l'exécuter
```

### Logs Détaillés

Mode verbose pour debugging :
```bash
python scripts/setup_database.py --verbose
# Affiche toutes les requêtes SQL exécutées

./scripts/deploy.sh production --verbose
# Affiche tous les détails Docker, migrations, etc.
```

## 📚 Bonnes Pratiques

### 1. Toujours Backup Avant Opération Destructrice
```bash
# ✓ BON
python scripts/backup_database.py
python scripts/setup_database.py --replace  # Opération destructrice

# ✗ MAUVAIS
python scripts/setup_database.py --replace  # Sans backup
```

### 2. Tester Scripts en Staging Avant Production
```bash
# 1. Test staging
./scripts/deploy.sh staging
# Vérifier application fonctionne

# 2. Si OK, déployer production
./scripts/deploy.sh production
```

### 3. Automatiser Backups (Cron)
```bash
# /etc/cron.d/saisie-auto-backup
# Backup quotidien 2h du matin
0 2 * * * ubuntu cd /opt/saisie-auto && python scripts/backup_database.py --upload-s3

# Backup hebdomadaire complet (dimanche 3h)
0 3 * * 0 ubuntu cd /opt/saisie-auto && python scripts/backup_database.py --upload-s3 --full
```

### 4. Versionner Scripts Avec Code

Scripts font partie du code, versionnés dans Git. Changements scripts = Pull Request avec review.

## 📞 Support

**Erreurs fréquentes :**

**"Connection refused" (BDD) :**
```bash
# Vérifier PostgreSQL tourne
systemctl status postgresql

# Vérifier DATABASE_URL correct
echo $DATABASE_URL
```

**"Permission denied" (backup) :**
```bash
# Vérifier permissions dossier backups
ls -ld backups/
chmod 755 backups/
```

**"Migration failed" :**
```bash
# Vérifier état migrations
cd backend
alembic current
alembic history

# Forcer revision spécifique si besoin
alembic stamp head
```

## ❓ FAQ

**Q : Peut-on annuler setup_database.py ?**  
R : Oui, si erreur pendant exécution, rollback automatique. Sinon, restore depuis backup : `psql < backup.sql`

**Q : Backups chiffrés ?**  
R : Pas par défaut. Ajouter chiffrement : `gpg -c backup.sql.gz`. Ou upload S3 avec chiffrement activé.

**Q : Déploiement zero-downtime ?**  
R : Oui. `deploy.sh` utilise rolling restart (services redémarrés un par un). Users pas impactés.

**Q : Combien de backups garder ?**  
R : Par défaut 30 jours (configurable). Backups > 30j supprimés automatiquement.

---

*Scripts version : 0.1.0*  
*Testé sur : Ubuntu 22.04, macOS 14, Python 3.11*