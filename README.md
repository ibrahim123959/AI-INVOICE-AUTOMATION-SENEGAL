# 🧾 Saisie Automatique de Factures — Plateforme IA

Plateforme d’automatisation intelligente de la saisie comptable de factures, conçue pour assister les comptables et auditeurs dans l’extraction des données, la détection de doublons, la proposition d’imputation comptable et l’export vers des logiciels comptables comme **Sage Sari** grace au pouvoir de l'intelligence artificielle.

Le système repose sur une architecture modulaire et explicable, où l’intelligence artificielle **assiste** l’utilisateur sans jamais remplacer la validation humaine.

Aperçu de la plateforme   👉👉   https://color-guitar-58033240.figma.site/
---

## 🎯 Contexte & Objectif du Projet


Ce projet est développé dans le cadre d’un **pilote institutionnel** avec :

- **SCC — Société de la Cour des Comptes (Sénégal)**
- **PGS — PanAfrican Gateway Solutions (Sénégal)**

Objectifs principaux :
- Réduire le temps de saisie manuelle des factures
- Limiter les erreurs comptables et les doublons
- Garantir la traçabilité et la conformité audit
- Préparer une industrialisation future multi-organisations

⚠️ Ce dépôt correspond à une **plateforme pilote**.  
Certaines fonctionnalités sont conceptuelles ou en cours de structuration.

---

## 🏗️ Architecture Générale

Le projet adopte une architecture **modulaire et évolutive**, pensée pour séparer clairement les responsabilités.

### 🧠 Backend Métier
- **Python + FastAPI**
- Orchestration du traitement des factures
- Gestion des règles métier comptables
- API REST versionnée (`/api/v1`)
- Journalisation complète (audit trail)

### 🤖 Moteur IA (ML Engine)
- Isolé du backend principal
- Basé sur **Sentence Transformers**
- Apprentissage supervisé à partir des validations humaines
- Moteur réutilisable pour d’autres cas d’usage futurs

### 📄 OCR — Extraction de données
- Abstraction multi-fournisseurs :
  - Google Document AI
  - AWS Textract
  - PaddleOCR (local)
- Possibilité de changer de fournisseur sans modifier le backend

### 🖥️ Frontend
- **React + Vite + Tailwind CSS**
- Interfaces explicatives et orientées validation humaine
- Conçu pour un usage professionnel (audit, comptabilité)

### 🗄️ Base de Données
- **PostgreSQL**
- Stockage structuré des factures, extractions, imputations, validations et exports

---

## 🔍 Fonctionnalités Clés

- 📥 Import de factures (PDF, image, email — futur)
- 🔎 Extraction OCR intelligente
- ⚠️ Détection de doublons
- 🧮 Proposition d’imputation comptable (SYSCOHADA)
- ✅ Validation humaine obligatoire
- 📤 Export compatible **Sage Sari / FEC**
- 📜 Traçabilité complète pour audit

---

## 📚 Documentation

La documentation est organisée par audience :

- 📐 Technique : `/docs/technical`
- 📊 Métier & comptable : `/docs/business`
- 👤 Utilisateur : `/docs/user`
- 🔐 Gouvernance & conformité : `/docs/governance`

👉 Voir la documentation complète dans le dossier [`docs/`](docs/)

---

## 🚀 Quick Start (Développement)

```bash
# Cloner le dépôt
git clone https://github.com/ibrahim123959/AI-INVOICE-AUTOMATION-SENEGAL.git
cd AI-INVOICE-AUTOMATION-SENEGAL


# Lancement environnement local (à venir)
docker-compose up
