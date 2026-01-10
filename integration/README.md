# Integration - Connecteurs Systèmes Externes

## 📖 Contexte

La plateforme ne fonctionne pas isolée. Elle doit s'intégrer avec systèmes comptables existants (Sage Sari) et potentiellement d'autres sources futures (email, ERP). Ce module gère toutes ces intégrations externes.

**Actuellement implémenté : Export Sage (FEC)**  
**Futur : Email inbox, API Sage directe, connecteurs ERP**
```
┌────────────────────────────────────────┐
│  Backend (factures validées)           │
└──────────────┬─────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│   ► INTEGRATION ◄ (Ce module)            │
│                                          │
│  sage/         → Export vers Sage Sari   │
│  email/        → Récupération emails     │
│  (futurs: ERP, banques...)               │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│  Sage Sari (logiciel comptable client)   │
│  Import fichier FEC généré               │
└──────────────────────────────────────────┘
```

## 🏗️ Structure
integration/
│
├── sage/                        # Intégration Sage Sari
│   ├── fec_generator.py         # Génération fichiers FEC
│   │                            # Formate écritures selon norme DGFiP
│   │                            # Vérifie équilibre débits/crédits
│   ├── fec_validator.py         # Validation format FEC
│   │                            # Conforme spécContinue20:23officielle
│   │                            # Détecte erreurs structure
│   ├── sage_api_client.py       # Client API Sage (futur)
│   │                            # Import automatique via API
│   └── templates/
│       └── fec_template.txt     # Structure fichier FEC
│
├── email/                       # Intégration email (futur)
│   ├── imap_client.py           # Connexion IMAP boîte email
│   │                            # Récupère emails avec PJ PDF
│   └── attachment_extractor.py  # Extraction pièces jointes
│                                # Filtre PDFs, envoie vers backend
│
└── tests/
    ├── test_fec_generation.py   # Tests génération FEC    
    └── test_fec_validation.py   # Tests validation format

## 📄 Format FEC (Fichier Échange Comptable)

### Qu'est-ce que le FEC ?

Le FEC est un format standardisé obligatoire en France (et adopté dans certains pays africains) pour exports comptables. Il permet à l'administration fiscale et aux auditeurs de vérifier la comptabilité.

**Structure : CSV avec pipe `|` comme séparateur**
JournalCode|JournalLib|EcritureNum|EcritureDate|CompteNum|CompteLib|CompAuxNum|CompAuxLib|PieceRef|PieceDate|EcritureLib|Debit|Credit|EcritureLet|DateLet|ValidDate|Montantdevise|Idevise

**Exemple facture SENELEC 177,000 FCFA :**
ACH|Achats|001|20241115|6054|Électricité|||SENELEC|FACT-2024-11-00456|20241115|Conso élec nov|150000.00|0.00|||||
ACH|Achats|001|20241115|44566|TVA déductible|||SENELEC|FACT-2024-11-00456|20241115|TVA 18%|27000.00|0.00|||||
ACH|Achats|001|20241115|401|Fournisseurs|SENELEC|SENELEC SARL|FACT-2024-11-00456|20241115|Fact SENELEC|0.00|177000.00|||||

**Règles strictes :**
- Équilibre : Total débits = Total crédits (par écriture)
- Dates format YYYYMMDD
- Montants format décimal avec `.00`
- Pas d'espaces dans montants
- Séparateur `|` obligatoire

### Pourquoi Important pour SCC/Audit ?

La Cour des Comptes et auditeurs demandent souvent FEC pour vérifier comptes. Un FEC invalide = audit bloqué.

Notre générateur garantit conformité 100%.

## 🔧 Usage

### Génération FEC
```python
from integration.sage import FECGenerator

# Initialise générateur
generator = FECGenerator()

# Récupère factures validées depuis BDD
validated_invoices = db.query(Invoice).filter_by(status="validated").all()

# Génère FEC
fec_content = generator.generate(
    invoices=validated_invoices,
    journal_code="ACH",      # Code journal (Achats)
    journal_label="Achats",
    period_start="2024-11-01",
    period_end="2024-11-30"
)

# Sauvegarde fichier
with open("FEC_202411.txt", "w", encoding="utf-8") as f:
    f.write(fec_content)
```

### Validation FEC
```python
from integration.sage import FECValidator

validator = FECValidator()

# Valide fichier généré
result = validator.validate("FEC_202411.txt")

if result.is_valid:
    print("✓ FEC valide, prêt pour Sage")
else:
    print("✗ Erreurs détectées :")
    for error in result.errors:
        print(f"  - Ligne {error.line}: {error.message}")
```

**Validations effectuées :**
- Structure colonnes (18 colonnes obligatoires)
- Format dates (YYYYMMDD)
- Format montants (décimal, pas d'espace)
- Équilibre débits/crédits par écriture
- Cohérence CompteNum (existe dans plan comptable)
- Pas de lignes vides
- Encoding UTF-8

## 🚀 Workflow Complet Backend → Sage

**Étape 1 : Utilisateur valide toutes factures du mois**

Backend : Toutes factures statut `validated`

**Étape 2 : Click "Exporter vers Sage" (frontend)**
```javascript
// Frontend
const response = await exportService.generateFEC({
  period: "2024-11",
  format: "sage"
});

// Backend reçoit requête
POST /v1/exports/fec
Body: { "period": "2024-11", "format": "sage" }
```

**Étape 3 : Backend génère FEC**
```python
# backend/app/api/v1/exports.py
@router.post("/fec")
async def generate_fec_export(request: FECExportRequest):
    # Récupère factures validées période
    invoices = await get_validated_invoices(request.period)
    
    # Génère FEC via integration/sage
    from integration.sage import FECGenerator
    generator = FECGenerator()
    fec_content = generator.generate(invoices)
    
    # Valide
    from integration.sage import FECValidator
    validator = FECValidator()
    validation = validator.validate_content(fec_content)
    
    if not validation.is_valid:
        raise HTTPException(400, detail=validation.errors)
    
    # Sauvegarde fichier
    filename = f"FEC_{request.period}.txt"
    file_path = save_export(filename, fec_content)
    
    # Enregistre historique
    export_record = SageExport(
        period=request.period,
        filename=filename,
        file_path=file_path,
        invoice_count=len(invoices)
    )
    db.add(export_record)
    await db.commit()
    
    return {"export_id": export_record.id, "filename": filename}
```

**Étape 4 : Frontend télécharge fichier**
```javascript
// Frontend
const blob = await exportService.downloadFEC(exportId);
downloadFile(blob, "FEC_202411.txt");
```

**Étape 5 : Utilisateur importe dans Sage**

1. Ouvre Sage Sari
2. Menu "Fichier" → "Importer" → "FEC"
3. Sélectionne `FEC_202411.txt`
4. Sage valide format, importe écritures
5. Écritures apparaissent dans journaux Sage

## 📧 Intégration Email (Futur)

**Use case :** Fournisseurs envoient factures par email. Automatiser réception.

**Architecture prévue :**
```python
# integration/email/imap_client.py
class EmailInvoiceCollector:
    def __init__(self, email, password, imap_server):
        self.email = email
        self.imap = connect_imap(imap_server, email, password)
    
    async def fetch_new_invoices(self):
        """Récupère emails non lus avec PJ PDF"""
        emails = self.imap.search("UNSEEN")
        
        invoices = []
        for email in emails:
            attachments = extract_attachments(email)
            pdf_attachments = [a for a in attachments if a.endswith('.pdf')]
            
            for pdf in pdf_attachments:
                # Upload vers backend
                response = await backend_api.upload_invoice(pdf)
                invoices.append(response)
        
        return invoices

# Cron job (toutes les heures)
collector = EmailInvoiceCollector("factures@entreprise.sn", "pass", "imap.gmail.com")
new_invoices = await collector.fetch_new_invoices()
```

**Bénéfices :**
- Zéro action manuelle
- Factures traitées dès réception email
- Réduction délai traitement de jours → heures

## 🧪 Tests

### Tests Génération FEC
```python
# tests/test_fec_generation.py
def test_fec_format_valid():
    """Vérifie FEC généré respecte format"""
    generator = FECGenerator()
    
    # Données test
    invoices = [create_test_invoice(supplier="SENELEC", amount=177000)]
    
    fec = generator.generate(invoices)
    
    # Vérifie structure
    lines = fec.strip().split('\n')
    assert len(lines) == 4  # Header + 3 lignes écriture
    
    # Vérifie header
    assert lines[0].startswith("JournalCode|JournalLib|")
    
    # Vérifie débits = crédits
    debits = sum(extract_debit(line) for line in lines[1:])
    credits = sum(extract_credit(line) for line in lines[1:])
    assert debits == credits == 177000.00
```

### Tests Validation
```python
def test_validator_detects_unbalanced():
    """Vérifie détection déséquilibre"""
    validator = FECValidator()
    
    # FEC invalide (débits ≠ crédits)
    invalid_fec = """
JournalCode|JournalLib|...
ACH|Achats|001|...|150000.00|0.00|...
ACH|Achats|001|...|0.00|100000.00|...
    """
    
    result = validator.validate_content(invalid_fec)
    
    assert result.is_valid == False
    assert any("déséquilibre" in err.message for err in result.errors)
```

## 📚 Ressources

### Format FEC
- [Spécification DGFiP (France)](https://www.impots.gouv.fr/portail/files/media/1_metier/2_professionnel/EV/2_gestion/270_fec/fec_2013.pdf)
- [Guide FEC Sage](https://www.sage.com/fr-fr/blog/export-fec/)

### IMAP Python
- [imaplib documentation](https://docs.python.org/3/library/imaplib.html)
- [Email parsing](https://docs.python.org/3/library/email.html)

## ❓ FAQ

**Q : FEC obligatoire pour SCC ?**  
R : Pas strictement, mais standard comptable reconnu. Facilite import dans Sage et audits.

**Q : Sage API disponible ?**  
R : Dépend version Sage. API existe (Sage X3, Sage 100), pas toujours activée. Export FEC = fallback universel.

**Q : Peut-on exporter vers QuickBooks/Tompro ?**  
R : Oui, créer adaptateurs similaires. QuickBooks = CSV différent, Tompro = format propriétaire (à documenter).

**Q : Email collector sécurisé ?**  
R : Oui. Credentials chiffrés, emails supprimés après traitement, logs anonymisés.

---

*Version : 0.1.0*  
*Intégrations : Sage (FEC), Email (en développement)*