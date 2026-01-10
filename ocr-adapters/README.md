# OCR Adapters - Abstraction Multi-Vendors

## 📖 Contexte

Les factures arrivent sous différents formats : PDF natifs (générés électroniquement), PDF scannés, photos prises au smartphone. Pour extraire les données (fournisseur, date, montants), nous utilisons des technologies OCR (Optical Character Recognition - Reconnaissance Optique de Caractères).

Le problème : chaque fournisseur OCR (Google, AWS, PaddleOCR) a sa propre API, ses formats de réponse, ses particularités. Si nous codions directement contre l'API Google dans le backend, nous serions prisonniers de ce choix. Changer de fournisseur nécessiterait de réécrire du code partout.

**Solution : Pattern Adapter**

Ce module implémente une couche d'abstraction. Peu importe le fournisseur utilisé en arrière-plan, le backend reçoit toujours la même structure de données.
```
Backend appelle :
  ocr_service.extract("facture.pdf")
    ↓
OCR Adapters (ce module) décide selon config :
  ├── GoogleDocumentAIAdapter.extract()  (si config = "google")
  ├── AWSTextractAdapter.extract()       (si config = "textract")
  └── PaddleOCRAdapter.extract()         (si config = "paddle")
    ↓
Tous retournent même format :
  ExtractionResult {
    supplier_name, invoice_date, amounts, line_items, confidence
  }
```

## 🏗️ Architecture
```
ocr-adapters/
│
├── base_adapter.py              # Interface abstraite (contrat)
│                                # Définit méthode extract() que tous doivent implémenter
│
├── google_document_ai.py        # Implémentation Google Document AI
│                                # Précision : 95-98%
│                                # Coût : $1.50/1000 pages
│                                # Meilleur sur factures françaises
│
├── aws_textract.py              # Implémentation AWS Textract
│                                # Précision : 93-96%
│                                # Coût : $1.50/1000 pages
│                                # Excellent sur tableaux complexes
│
├── paddle_ocr.py                # Implémentation PaddleOCR (local)
│                                # Précision : 90-93%
│                                # Coût : Gratuit
│                                # Aucune dépendance externe
│
├── config/                      # Configurations spécifiques
│   ├── google_config.json       # Paramètres Google (zones intérêt, langue)
│   ├── textract_config.json     # Paramètres AWS
│   └── paddle_config.json       # Modèles PaddleOCR, langues supportées
│
└── tests/                       # Tests adapters
    ├── test_google_adapter.py   # Tests spécifiques Google
    ├── test_textract_adapter.py # Tests spécifiques AWS
    ├── test_paddle_adapter.py   # Tests spécifiques Paddle
    └── test_adapter_contract.py # Vérifie tous respectent interface
```

## 📐 Interface Commune (Contrat)

Tous les adapters implémentent cette interface :
```python
class BaseAdapter(ABC):
    @abstractmethod
    def extract(self, file_path: str) -> ExtractionResult:
        """
        Extrait données structurées depuis facture.
        
        Args:
            file_path: Chemin vers fichier PDF/image
            
        Returns:
            ExtractionResult contenant :
            - supplier_name (str)
            - supplier_normalized (str)  # "SENELEC SA" → "SENELEC"
            - invoice_number (str)
            - invoice_date (date)
            - amount_ht (Decimal)
            - amount_tva (Decimal)
            - amount_ttc (Decimal)
            - tva_rate (Decimal)
            - description (str)
            - line_items (List[Dict])
            - confidence_score (float)  # 0.0-1.0
            
        Raises:
            OCRError: Si extraction échoue
        """
        pass
```

## 🔧 Comparaison Vendors

| Critère | Google Document AI | AWS Textract | PaddleOCR |
|---------|-------------------|--------------|-----------|
| **Précision factures FR** | ⭐⭐⭐⭐⭐ 95-98% | ⭐⭐⭐⭐ 93-96% | ⭐⭐⭐⭐ 90-93% |
| **Coût** | $1.50/1000 pages | $1.50/1000 pages | Gratuit |
| **Vitesse** | 1-3 sec/page | 2-4 sec/page | 1-2 sec/page |
| **Extraction structurée** | Native (JSON) | Native (JSON) | Manuelle |
| **Dépendance externe** | Oui (API Google) | Oui (API AWS) | Non (self-hosted) |
| **Données locales** | Non (envoyées cloud) | Non (envoyées cloud) | Oui (100% local) |
| **Langues** | 100+ | 50+ | 80+ |
| **Setup** | Compte GCP + API key | Compte AWS + credentials | pip install |

## 🚀 Usage

### Configuration (backend/app/config.py)
```python
# Changer fournisseur = 1 ligne
OCR_PROVIDER = "google"  # ou "textract" ou "paddle"
```

### Code Backend
```python
from ocr_adapters import get_adapter

# Obtient adapter selon config
adapter = get_adapter(config.OCR_PROVIDER)

# Extrait données (même code peu importe vendor)
result = adapter.extract("uploads/facture_senelec.pdf")

# Utilise résultat
print(f"Fournisseur : {result.supplier_name}")
print(f"Montant TTC : {result.amount_ttc} FCFA")
print(f"Confiance : {result.confidence_score * 100}%")
```

## 🔄 Changement de Vendor (Sans Modifier Backend)

**Scenario : Google trop cher, switch vers PaddleOCR**
```bash
# 1. Modifier config
# backend/app/config.py
OCR_PROVIDER = "paddle"  # était "google"

# 2. Redémarrer backend
docker-compose restart backend

# C'est tout. Backend utilise maintenant PaddleOCR.
```

## 📦 Ajout Nouveau Vendor

**Exemple : Ajouter Tesseract OCR**
```python
# 1. Créer tesseract_adapter.py
from ocr_adapters.base_adapter import BaseAdapter, ExtractionResult
import pytesseract

class TesseractAdapter(BaseAdapter):
    def extract(self, file_path: str) -> ExtractionResult:
        # Implémentation Tesseract
        text = pytesseract.image_to_string(file_path, lang='fra')
        # Parse text, extrait données...
        return ExtractionResult(...)

# 2. Enregistrer dans factory
# ocr_adapters/__init__.py
ADAPTERS = {
    "google": GoogleDocumentAIAdapter,
    "textract": AWSTextractAdapter,
    "paddle": PaddleOCRAdapter,
    "tesseract": TesseractAdapter  # Nouveau
}

# 3. Utiliser
OCR_PROVIDER = "tesseract"
```

## 🧪 Tests

### Tests Conformité Interface

Vérifie que tous adapters respectent contrat :
```bash
pytest tests/test_adapter_contract.py -v

# Teste :
# - Méthode extract() existe
# - Retourne ExtractionResult
# - Gère erreurs correctement
# - Tous champs obligatoires présents
```

### Tests Spécifiques Vendor
```bash
# Google
pytest tests/test_google_adapter.py -v

# AWS
pytest tests/test_textract_adapter.py -v

# Paddle
pytest tests/test_paddle_adapter.py -v
```

### Tests Précision (avec factures réelles)
```python
# tests/test_precision.py
def test_senelec_invoice_extraction():
    """Teste extraction facture SENELEC avec résultat attendu"""
    adapter = get_adapter("google")
    
    result = adapter.extract("tests/fixtures/senelec_sample.pdf")
    
    assert result.supplier_name == "SENELEC"
    assert result.amount_ttc == 177000.00
    assert result.confidence_score > 0.90
```

## 🔒 Sécurité

**Secrets API (Google, AWS)**

Jamais dans code, toujours variables environnement :
```bash
# .env
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=secret...
```

**Données sensibles**

- Google/AWS : Factures envoyées vers cloud (chiffrement transit)
- PaddleOCR : Traitement 100% local, aucune donnée externe

Pour SCC (Cour des Comptes), PaddleOCR recommandé (données publiques sensibles).

## 📚 Ressources

### Google Document AI
- [Documentation](https://cloud.google.com/document-ai/docs)
- [Pricing](https://cloud.google.com/document-ai/pricing)
- [Setup guide](https://cloud.google.com/document-ai/docs/setup)

### AWS Textract
- [Documentation](https://docs.aws.amazon.com/textract/)
- [Pricing](https://aws.amazon.com/textract/pricing/)
- [Getting started](https://docs.aws.amazon.com/textract/latest/dg/getting-started.html)

### PaddleOCR
- [GitHub](https://github.com/PaddlePaddle/PaddleOCR)
- [Documentation](https://paddlepaddle.github.io/PaddleOCR/)
- [Models multilangues](https://github.com/PaddlePaddle/PaddleOCR/blob/release/2.7/doc/doc_en/models_list_en.md)

## ❓ FAQ

**Q : Pourquoi pas un seul OCR pour tout ?**  
R : Flexibilité. Google meilleur précision, Paddle gratuit, AWS bon sur tableaux. On choisit selon budget/besoin.

**Q : Performance si on change vendor ?**  
R : Tests montrent <5% différence précision entre Google et Paddle sur factures standards. Tableaux complexes : Google/AWS meilleurs.

**Q : Peut-on utiliser plusieurs vendors simultanément ?**  
R : Oui (futur). Stratégie : Paddle d'abord (gratuit), si confiance <80% → fallback Google (payant mais précis).

**Q : Adapter fonctionne avec images floues ?**  
R : Google/Textract gèrent mieux flou. Paddle nécessite images nettes. Prétraitement image recommandé (contraste, rotation).

---

*Version : 0.1.0*  
*Adapters implémentés : Google Document AI, AWS Textract, PaddleOCR*