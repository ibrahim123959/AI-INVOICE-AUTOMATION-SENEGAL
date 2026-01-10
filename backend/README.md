# Backend - Cœur de la Plateforme de Saisie Automatique

## 📖 Contexte et Vision d'Ensemble

Dans le processus traditionnel de comptabilité, la saisie manuelle des factures représente une charge de travail considérable : un comptable passe en moyenne 4-5 minutes par facture à lire, identifier les informations clés, décider du compte comptable approprié, puis saisir manuellement dans le logiciel comptable. Pour 50 factures par jour, cela représente plus de 4 heures de travail répétitif.

Notre plateforme automatise ce processus grâce à l'intelligence artificielle. Le **backend** est le cerveau central qui orchestre toute cette automatisation. Il reçoit les factures uploadées par l'utilisateur, coordonne leur traitement par différents systèmes spécialisés (reconnaissance optique de caractères, analyse sémantique), puis présente les résultats structurés à l'utilisateur pour validation.

**Place dans l'écosystème global :**
```
            ┌─────────────┐
            │  Utilisateur│ (Upload facture via navigateur)
            └──────┬──────┘
                   ↓
┌──────────────────────────────────────────────┐
│         FRONTEND (Interface Web)             │
└──────────────────┬───────────────────────────┘
                   ↓ 
              (API REST)
┌──────────────────────────────────────────────┐
│    ► BACKEND ◄ (Ce module - Orchestrateur)   │
│                                              │
│  Coordonne :                                 │
│  • OCR (extraction texte)                    │
│  • ML Engine (suggestion comptes)            │
│  • Base de données (persistance)             │
│  • Export Sage (génération FEC)              │
└──────────────────────────────────────────────┘
```

Le backend ne fait pas tout le travail lui-même - il délègue les tâches spécialisées à d'autres modules tout en gardant la responsabilité de la cohérence globale et de la logique métier comptable.

---

## 🏗️ Architecture Détaillée

Le backend est organisé en couches logiques claires, chacune ayant une responsabilité précise :

### 📂 Structure des Dossiers
```
backend/
│
├── app/                           # Code application principal
│   │
│   ├── main.py                    # Point d'entrée FastAPI
│   │                              # Configure serveur, routes, middleware
│   │                              # Lance au démarrage : uvicorn app.main:app
│   │
│   ├── config.py                  # Configuration centralisée
│   │                              # Variables environnement, paramètres OCR/ML
│   │                              # Lecture depuis .env
│   │
│   ├── api/                       # Couche exposition HTTP
│   │   ├── v1/                    # Version 1 de l'API (versioning futur)
│   │   │   ├── invoices.py        # Routes upload, consultation factures
│   │   │   │                      # POST /upload, GET /invoices/{id}
│   │   │   ├── validation.py      # Routes validation utilisateur
│   │   │   │                      # POST /validate/{id}, PATCH /correct/{id}
│   │   │   ├── exports.py         # Routes génération exports Sage
│   │   │   │                      # POST /export/fec, GET /export/{id}/download
│   │   │   └── admin.py           # Routes administration
│   │   │                          # GET /stats, GET /health
│   │   └── dependencies.py        # Dépendances injectées (auth, BDD session)
│   │
│   ├── core/                      # Logique métier centrale (le "cerveau")
│   │   ├── invoice_processor.py   # Orchestration complète traitement facture
│   │   │                          # Coordonne : OCR → extraction → ML → stockage
│   │   │                          # Gère états : uploaded → processing → completed
│   │   ├── duplicate_detector.py  # Détection intelligente doublons
│   │   │                          # Compare fournisseur, numéro, montant, date
│   │   │                          # Évite double paiement (critique pour audit)
│   │   ├── imputation_engine.py   # Moteur suggestion comptes comptables
│   │   │                          # Combine : règles métier + ML + historique
│   │   │                          # Retourne suggestions avec scores confiance
│   │   ├── learning_controller.py # Apprentissage supervisé depuis validations
│   │   │                          # Enregistre corrections utilisateur
│   │   │                          # Met à jour règles pour améliorer futur
│   │   └── audit_logger.py        # Logging exhaustif pour traçabilité
│   │                              # Enregistre qui, quoi, quand pour chaque action
│   │                              # Obligatoire pour conformité audit (SCC)
│   │
│   ├── models/                    # Modèles de données (ORM SQLAlchemy)
│   │   │                          # Représentent tables PostgreSQL
│   │   ├── invoice.py             # Table factures : métadonnées upload
│   │   │                          # Colonnes : id, filename, upload_date, status...
│   │   ├── extraction.py          # Table extractions : données OCR structurées
│   │   │                          # Colonnes : supplier, date, amounts, description...
│   │   ├── imputation.py          # Table imputations : suggestions + validations
│   │   │                          # Colonnes : suggested_account, confidence, validated...
│   │   ├── account_rule.py        # Table règles : mapping fournisseur → compte
│   │   │                          # Ex: SENELEC → toujours compte 6054
│   │   ├── chart_of_accounts.py   # Table plan comptable : tous comptes SYSCOHADA
│   │   │                          # Ex: 6054 - Électricité, 6261 - Téléphone...
│   │   ├── learning_event.py      # Table apprentissage : historique corrections
│   │   │                          # Trace évolution système (amélioration continue)
│   │   └── sage_export.py         # Table exports : historique FEC générés
│   │                              # Traçabilité exports vers Sage
│   │
│   ├── schemas/                   # Schémas Pydantic (validation entrées/sorties)
│   │   │                          # Valident données API (sécurité + cohérence)
│   │   ├── invoice_schema.py      # Schémas requêtes/réponses factures
│   │   │                          # Ex: InvoiceUploadRequest, InvoiceResponse
│   │   ├── extraction_schema.py   # Schémas données extraites
│   │   │                          # Ex: ExtractionResult (supplier, amounts...)
│   │   ├── imputation_schema.py   # Schémas suggestions comptables
│   │   │                          # Ex: ImputationSuggestion (account, confidence)
│   │   └── export_schema.py       # Schémas exports
│   │                              # Ex: FECExportRequest, ExportStatus
│   │
│   ├── services/                  # Services externes et abstractions
│   │   │                          # Isolent communication avec systèmes externes
│   │   ├── ocr_service.py         # Interface vers adaptateurs OCR
│   │   │                          # Appelle ocr-adapters/ selon config
│   │   │                          # Abstrait Google/AWS/Paddle derrière interface commune
│   │   ├── ml_service.py          # Client HTTP vers ml-engine
│   │   │                          # POST /encode, POST /match vers port 8001
│   │   │                          # Gère timeouts, retries, erreurs
│   │   ├── storage_service.py     # Gestion stockage fichiers
│   │   │                          # Local (uploads/) ou cloud (S3/GCS futur)
│   │   │                          # Archivage sécurisé factures originales
│   │   └── sage_connector.py      # Pont vers module integration/sage
│   │                              # Génère FEC, valide format, prépare export
│   │
│   ├── database/                  # Gestion base de données
│   │   ├── session.py             # Factory sessions SQLAlchemy
│   │   │                          # Crée connexions BDD, gère transactions
│   │   ├── migrations/            # Historique évolutions schéma BDD (Alembic)
│   │   │   └── versions/          # Fichiers migration versionnés
│   │   │                          # Ex: 001_create_invoices_table.py
│   │   │                          # Permet rollback si problème
│   │   └── seeds/                 # Données initiales à charger
│   │       └── syscohada_chart.sql # Plan comptable SYSCOHADA complet
│   │                              # ~500 comptes : 1xxx (bilan), 6xxx (charges)...
│   │
│   └── utils/                     # Utilitaires transverses
│       ├── validators.py          # Validateurs métier spécifiques
│       │                          # Ex: valider format NINEA, cohérence dates
│       ├── normalizers.py         # Normalisation données
│       │                          # Ex: "SENELEC SA" → "SENELEC"
│       │                          # "150 000,00" → 150000.00
│       ├── logger.py              # Configuration logging centralisé
│       │                          # Format structuré (JSON), niveaux (INFO/ERROR)
│       │                          # Sortie : console (dev), fichiers (prod)
│       └── exceptions.py          # Exceptions métier personnalisées
│                                  # Ex: DuplicateInvoiceError, InvalidAmountError
│
├── tests/                         # Suite de tests complète
│   ├── conftest.py                # Fixtures pytest partagées
│   │                              # Ex: client API test, BDD test isolée
│   ├── unit/                      # Tests unitaires (fonctions isolées)
│   │   ├── test_duplicate_detector.py  # Teste détection doublons
│   │   ├── test_imputation_engine.py   # Teste logique suggestions
│   │   └── test_normalizers.py         # Teste normalisation données
│   ├── integration/               # Tests intégration (flux complets)
│   │   ├── test_invoice_flow.py        # Upload → OCR → ML → Export
│   │   └── test_learning_flow.py       # Validation → Apprentissage
│   └── fixtures/                  # Données test réalistes
│       ├── sample_invoices/       # PDFs factures test (SENELEC, Orange...)
│       └── expected_extractions.json   # Résultats attendus (référence)
│
├── Dockerfile                     # Image Docker production
├── requirements.txt               # Dépendances Python (FastAPI, SQLAlchemy...)
├── pyproject.toml                 # Métadonnées projet (Poetry/pip moderne)
├── pytest.ini                     # Configuration tests
└── .env.example                   # Template variables environnement
```

---

## 🎯 Responsabilités Clés

### 1. **Orchestration du Flux de Traitement**

Le backend coordonne l'ensemble du processus de traitement d'une facture :

**Étape 1 - Réception (endpoint `/upload`)**
- Reçoit fichier PDF/image depuis frontend
- Valide format et taille (max 10 MB)
- Génère ID unique, stocke temporairement
- Crée entrée en BDD avec statut `uploaded`

**Étape 2 - Extraction OCR (via `ocr_service`)**
- Appelle adaptateur OCR configuré (Google/AWS/Paddle)
- Reçoit données structurées : fournisseur, date, montants
- Enregistre dans table `extractions`
- Statut BDD → `extracted`

**Étape 3 - Suggestion Imputation (via `ml_service` + `imputation_engine`)**
- Envoie description facture au ML engine
- Reçoit embeddings et suggestions comptes
- Applique règles métier (fournisseur connu → règle prioritaire)
- Calcule score confiance final
- Enregistre dans table `imputations`
- Statut BDD → `ready_for_validation`

**Étape 4 - Validation Utilisateur (endpoint `/validate`)**
- Présente suggestions à utilisateur via frontend
- Utilisateur valide ou corrige
- Si correction : enregistre dans `learning_events` pour apprentissage
- Statut BDD → `validated`

**Étape 5 - Export Sage (endpoint `/export/fec`)**
- Génère fichier FEC (Format Échange Comptable)
- Valide conformité format (équilibre débits/crédits)
- Propose téléchargement
- Enregistre historique export

### 2. **Application des Règles Métier Comptables**

Le backend encode les règles métier spécifiques à la comptabilité :

**Règle 1 : Équilibre comptable**
```
Total Débits = Total Crédits (toujours)
Facture SENELEC 177,000 FCFA :
  Débit 6054 (Électricité)      : 150,000
  Débit 44566 (TVA déductible)  :  27,000
  Crédit 401 (Fournisseur)      : 177,000
  ✓ 177,000 = 177,000
```

**Règle 2 : Cohérence TVA**
- Si TVA 18% → montant TVA = HT × 0.18
- Détecte incohérences (montant déclaré ≠ calculé)

**Règle 3 : Plan comptable SYSCOHADA**
- Comptes 6xxx = Charges (dépenses)
- Comptes 401x = Fournisseurs
- Comptes 44xxx = TVA
- Validation : compte proposé existe dans référentiel

**Règle 4 : Détection doublons**
- Même fournisseur + même numéro facture = doublon probable
- Même montant + date proche (±30 jours) = suspect
- Alerte utilisateur avant validation

### 3. **Traçabilité et Audit Trail**

Pour la Cour des Comptes et institutions d'audit, chaque action doit être traçable :

**Ce qui est tracé :**
- Qui a uploadé la facture (user_id, timestamp)
- Quelle version OCR utilisée (provider, confidence)
- Quel modèle ML a suggéré (model_version, score)
- Qui a validé (user_id, timestamp)
- Si modification : valeur avant/après
- Export : qui, quand, quel fichier généré

**Pourquoi c'est critique :**
En cas d'audit (contrôle fiscal, inspection), on peut :
- Remonter de n'importe quelle écriture Sage à la facture source
- Prouver que validation humaine a eu lieu (IA assistive, pas autonome)
- Justifier pourquoi tel compte a été choisi
- Démontrer amélioration continue système (learning events)

### 4. **Gestion des Erreurs et Résilience**

Le backend gère gracieusement les échecs potentiels :

**OCR échoue (PDF illisible, format bizarre)**
→ Statut `extraction_failed`, notifie utilisateur, propose upload manuel données

**ML engine indisponible (service down, timeout)**
→ Fallback sur règles métier seules, réduit confiance mais permet continuation

**Doublon détecté**
→ Bloque validation automatique, force revue manuelle utilisateur

**Export FEC invalide (déséquilibre)**
→ Refuse génération, affiche détail erreur comptable, demande correction

---

## 🚀 Démarrage Rapide

### Prérequis

- **Python 3.11+** ([Télécharger Python](https://www.python.org/downloads/))
- **PostgreSQL 15+** ([Télécharger PostgreSQL](https://www.postgresql.org/download/))
- **Git** ([Installer Git](https://git-scm.com/downloads))

### Installation Locale (Développement)
```bash
# 1. Naviguer vers dossier backend
cd backend

# 2. Créer environnement virtuel Python (isolement dépendances)
python3 -m venv venv

# 3. Activer environnement
# Sur macOS/Linux :
source venv/bin/activate
# Sur Windows :
venv\Scripts\activate

# 4. Installer toutes les dépendances
pip install -r requirements.txt

# 5. Configurer variables environnement
cp .env.example .env
# Éditer .env avec vos valeurs (DATABASE_URL, etc.)

# 6. Initialiser base de données
python -m alembic upgrade head  # Applique migrations
python ../scripts/load_chart_of_accounts.py  # Charge plan comptable

# 7. Lancer serveur développement
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**API disponible sur :** `http://localhost:8000`  
**Documentation interactive (Swagger) :** `http://localhost:8000/docs`  
**Documentation alternative (ReDoc) :** `http://localhost:8000/redoc`

### Démarrage avec Docker (Recommandé)
```bash
# Depuis racine projet
docker-compose up backend

# Accès identique : http://localhost:8000
```

Avantage Docker : PostgreSQL, backend, ml-engine démarrent ensemble automatiquement.

---

## 🔗 Interactions avec Autres Modules

### Avec `ocr-adapters/`

**Communication :** Import Python direct (même processus)
```python
from ocr_adapters import get_adapter

adapter = get_adapter("google")  # ou "textract", "paddle"
result = adapter.extract("facture.pdf")
# → result contient données structurées
```

**Données reçues :**
```json
{
  "supplier_name": "SENELEC",
  "invoice_date": "2024-11-15",
  "amount_ht": 150000.00,
  "amount_tva": 27000.00,
  "amount_ttc": 177000.00,
  "confidence": 0.96
}
```

### Avec `ml-engine/`

**Communication :** HTTP REST (services séparés)
```python
import httpx

response = httpx.post(
    "http://localhost:8001/match",
    json={
        "description": "Consommation électrique novembre",
        "supplier": "SENELEC"
    }
)
suggestions = response.json()
# → suggestions = [{"account": "6054", "confidence": 0.92}, ...]
```

**Pourquoi HTTP et pas import direct ?**
- ML engine peut tourner sur machine différente (avec GPU)
- Permet scaling indépendant (plus d'instances ML si charge)
- Isolation : crash ML n'affecte pas backend

### Avec `frontend/`

**Communication :** API REST (HTTP JSON)

Frontend envoie requêtes, backend répond en JSON structuré.

**Exemple upload facture :**
```javascript
// Frontend (JavaScript)
const formData = new FormData();
formData.append('file', pdfFile);

const response = await fetch('http://localhost:8000/v1/invoices/upload', {
  method: 'POST',
  body: formData
});

const result = await response.json();
// → { "invoice_id": "uuid-123", "status": "processing" }
```

### Avec `integration/sage/`

**Communication :** Import Python direct
```python
from integration.sage import FECGenerator

generator = FECGenerator()
fec_content = generator.generate(validated_invoices)
# → fec_content = contenu fichier FEC prêt pour Sage
```

---

## 🧪 Tests

### Tests Unitaires

Testent fonctions isolées (logique métier pure) :
```bash
# Tous les tests unitaires
pytest tests/unit/ -v

# Test spécifique
pytest tests/unit/test_duplicate_detector.py -v

# Avec couverture code
pytest tests/unit/ --cov=app --cov-report=html
# Ouvre htmlcov/index.html pour rapport détaillé
```

**Exemple test unitaire :**
```python
# tests/unit/test_duplicate_detector.py
def test_detect_exact_duplicate():
    """Vérifie détection doublon exact (même fournisseur, numéro, montant)"""
    detector = DuplicateDetector()
    
    invoice1 = {"supplier": "SENELEC", "number": "F-2024-001", "amount": 150000}
    invoice2 = {"supplier": "SENELEC", "number": "F-2024-001", "amount": 150000}
    
    result = detector.check(invoice1, invoice2)
    
    assert result.is_duplicate == True
    assert result.confidence > 0.95
```

### Tests d'Intégration

Testent flux complets (plusieurs modules ensemble) :
```bash
# Tous les tests intégration
pytest tests/integration/ -v

# Test flux upload → extraction → suggestion
pytest tests/integration/test_invoice_flow.py -v
```

**Exemple test intégration :**
```python
# tests/integration/test_invoice_flow.py
def test_complete_invoice_processing(client, db):
    """Teste flux complet : upload PDF → OCR → ML → validation"""
    # Upload facture test
    response = client.post("/v1/invoices/upload", files={"file": sample_pdf})
    assert response.status_code == 200
    invoice_id = response.json()["invoice_id"]
    
    # Attendre traitement (OCR + ML)
    time.sleep(2)
    
    # Vérifier extraction réussie
    extraction = db.query(Extraction).filter_by(invoice_id=invoice_id).first()
    assert extraction.supplier_name == "SENELEC"
    assert extraction.amount_ttc == 177000.00
    
    # Vérifier suggestion imputation
    imputation = db.query(Imputation).filter_by(invoice_id=invoice_id).first()
    assert imputation.suggested_account == "6054"
    assert imputation.confidence > 0.80
```

### Couverture de Code

Objectif : **minimum 80% du code testé**
```bash
# Générer rapport couverture
pytest tests/ --cov=app --cov-report=term --cov-report=html

# Affiche dans terminal :
# ----------- coverage: platform darwin, python 3.11.5 -----------
# Name                              Stmts   Miss  Cover
# -----------------------------------------------------
# app/core/duplicate_detector.py       45      3    93%
# app/core/imputation_engine.py        67      8    88%
# app/services/ocr_service.py          34      5    85%
# -----------------------------------------------------
# TOTAL                               512     41    92%
```

---

## 📝 Conventions de Code

### Style Python (PEP 8)
```python
# ✅ BON : Type hints, docstring, nommage clair
async def process_invoice(
    invoice_id: str,
    ocr_provider: str = "google"
) -> InvoiceProcessingResult:
    """
    Traite une facture uploadée : extraction OCR + suggestion imputation.
    
    Args:
        invoice_id: UUID de la facture à traiter
        ocr_provider: Provider OCR à utiliser ("google", "textract", "paddle")
        
    Returns:
        InvoiceProcessingResult contenant extraction et suggestions
        
    Raises:
        OCRError: Si extraction échoue
        MLServiceUnavailable: Si ML engine injoignable
    """
    # Récupère facture depuis BDD
    invoice = await get_invoice(invoice_id)
    
    # Extrait données via OCR
    extraction = await ocr_service.extract(invoice.file_path, ocr_provider)
    
    # Obtient suggestions ML
    suggestions = await ml_service.suggest_accounts(extraction.description)
    
    return InvoiceProcessingResult(extraction=extraction, suggestions=suggestions)

# ❌ MAUVAIS : Pas de types, pas de docstring, nom vague
def proc(id, prov="g"):
    inv = get(id)
    ext = ocr(inv.fp, prov)
    sug = ml(ext.d)
    return res(ext, sug)
```

### Structure Asynchrone

FastAPI supporte async/await pour I/O non-bloquant :
```python
# ✅ BON : async pour opérations I/O (BDD, HTTP, fichiers)
@router.post("/upload")
async def upload_invoice(file: UploadFile, db: AsyncSession):
    # Sauvegarde fichier (I/O)
    file_path = await storage_service.save(file)
    
    # Insertion BDD (I/O)
    invoice = Invoice(filename=file.filename, file_path=file_path)
    db.add(invoice)
    await db.commit()
    
    # Appel OCR externe (I/O réseau)
    extraction = await ocr_service.extract(file_path)
    
    return {"invoice_id": invoice.id}

# ❌ MAUVAIS : Fonctions sync bloquent le serveur
@router.post("/upload")
def upload_invoice_sync(file: UploadFile, db: Session):
    # Ces appels bloquent, empêchent traiter d'autres requêtes
    file_path = storage_service.save_sync(file)  # Bloque
    db.add(invoice)
    db.commit()  # Bloque
    extraction = ocr_service.extract_sync(file_path)  # Bloque
```

**Pourquoi c'est important :**
- Avec async : serveur traite 100+ requêtes simultanément
- Avec sync : serveur traite 1 requête à la fois (lent si traitement long)

### Organisation Imports
```python
# ✅ BON : Groupés et triés
# 1. Standard library
import os
from datetime import datetime
from typing import Optional

# 2. Third-party
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

# 3. Application locale
from app.core.duplicate_detector import DuplicateDetector
from app.models.invoice import Invoice
from app.schemas.invoice_schema import InvoiceResponse

# ❌ MAUVAIS : Mélangé, non trié
from app.models.invoice import Invoice
import os
from fastapi import APIRouter
from app.core.duplicate_detector import DuplicateDetector
from datetime import datetime
```

---

## 🔐 Sécurité et Bonnes Pratiques

### Variables Sensibles

**JAMAIS** commiter secrets dans Git :
```python
# ❌ MAUVAIS : Secret en dur dans code
DATABASE_URL = "postgresql://user:password123@localhost/db"

# ✅ BON : Variable environnement
import os
DATABASE_URL = os.getenv("DATABASE_URL")
```

Fichier `.env` (gitignored) :
```env
DATABASE_URL=postgresql://user:password@localhost/saisie_auto
OCR_GOOGLE_API_KEY=AIza...
SECRET_KEY=your-secret-key-here
```

### Validation Entrées Utilisateur

Toujours valider et nettoyer inputs :
```python
from pydantic import BaseModel, validator

class InvoiceUpload(BaseModel):
    filename: str
    
    @validator('filename')
    def validate_filename(cls, v):
        # Interdit caractères dangereux (path traversal)
        if '..' in v or '/' in v or '\\' in v:
            raise ValueError("Nom fichier invalide")
        # Limite extensions autorisées
        if not v.lower().endswith(('.pdf', '.jpg', '.png')):
            raise ValueError("Format fichier non supporté")
        return v
```

### Logs Structurés (Pas de Données Sensibles)
```python
# ✅ BON : Log sans données personnelles
logger.info("Invoice uploaded", extra={
    "invoice_id": invoice.id,
    "filename": invoice.filename,
    "user_id": user.id  # ID, pas nom/email
})

# ❌ MAUVAIS : Log données sensibles
logger.info(f"Invoice from {supplier.name} - {supplier.email} - Amount {amount}")
```

---

## 📚 Ressources et Documentation

### FastAPI (Framework Backend)
- [Documentation officielle](https://fastapi.tiangolo.com/)
- [Tutorial débutant](https://fastapi.tiangolo.com/tutorial/)
- [Guide async/await Python](https://realpython.com/async-io-python/)

### SQLAlchemy (ORM Base de Données)
- [Documentation SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/)
- [Guide async SQLAlchemy](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Tutorial migrations Alembic](https://alembic.sqlalchemy.org/en/latest/tutorial.html)

### Pydantic (Validation Données)
- [Documentation Pydantic](https://docs.pydantic.dev/)
- [Guide validation complexe](https://docs.pydantic.dev/latest/usage/validators/)

### PostgreSQL
- [Documentation PostgreSQL](https://www.postgresql.org/docs/)
- [Guide indexation performance](https://www.postgresql.org/docs/current/indexes.html)

### Tests avec Pytest
- [Documentation Pytest](https://docs.pytest.org/)
- [Guide fixtures](https://docs.pytest.org/en/stable/fixture.html)
- [Pytest-asyncio (tests async)](https://pytest-asyncio.readthedocs.io/)

---

## 🤝 Contribution

Avant de contribuer au backend :

1. **Lire** [CONTRIBUTING.md](../CONTRIBUTING.md) à la racine
2. **Créer branche** depuis `develop` : `git checkout -b feature/ma-feature`
3. **Écrire tests** pour nouvelle fonctionnalité
4. **Vérifier couverture** : `pytest --cov=app`
5. **Linter** : `flake8 app/` (pas d'erreurs)
6. **Commit** : message clair (`feat: add duplicate detection`)
7. **Push** et créer **Pull Request** vers `develop`

---

## ❓ FAQ

**Q : Pourquoi FastAPI et pas Flask/Django ?**  
R : FastAPI est moderne (async natif), rapide (performances comparables à Node.js), auto-documente l'API (Swagger intégré), et a validation automatique (Pydantic).

**Q : Pourquoi PostgreSQL et pas MongoDB ?**
R : Données comptables sont relationnelles (factures ↔ extractions ↔ imputations). PostgreSQL garantit intégrité référentielle (FOREIGN KEY) et supporte transactions ACID (critique pour cohérence comptable). MongoDB mieux pour données non structurées.

**Q : Que se passe-t-il si OCR échoue ?**
R : Backend capture exception, marque facture extraction_failed, notifie frontend. Utilisateur peut soit réuploader meilleure qualité, soit saisir manuellement données.

**Q : Comment ajouter un nouveau fournisseur dans règles ?**
R : Insérer dans table account_rules : INSERT INTO account_rules (supplier_name, account_code) VALUES ('NOUVEAU_FOURNISSEUR', '6xxx'). Ou via interface admin (futur).

**Q : Backend peut traiter combien de factures simultanément ?**
R : Avec async, facilement 50-100 requêtes upload simultanées. Goulot d'étranglement = OCR externe (quotas API) ou ML engine (CPU/GPU). Solution : queue (Celery/Redis) pour traitement asynchrone en arrière-plan.

📞 Contact et Support
Questions techniques backend :

      Ouvrir issue GitHub avec tag backend
      Ou contacter équipe : [ibrahima4234@gmail.com]

Bugs ou comportements inattendus :

      Template bug report : .github/ISSUE_TEMPLATE/bug_report.md
      Inclure logs backend (dans logs/ ou console)