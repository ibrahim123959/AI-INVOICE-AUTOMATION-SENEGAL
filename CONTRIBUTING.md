
### **Fichier: CONTRIBUTING.md**

### Guide de Contribution - Saisie Auto AI

**Merci de contribuer à Saisie Auto AI ! Ce guide explique comment travailler efficacement sur le projet.**

## 🚀 Setup Initial

### 1. Fork et Clone

```bash
# Fork sur GitHub puis clone
git clone https://github.com/ibrahim123959/AI-INVOICE-AUTOMATION-SENEGAL.git
cd AI-INVOICE-AUTOMATION-SENEGAL
```

### 2. Install Dependencies

**Backend:**
```bash
cd backend
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**ML Engine:**
```bash
cd ml-engine
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Frontend:**
```bash
cd frontend
npm install
```

### 3. Configure Environment
```bash
cp backend/.env.example backend/.env
cp ml-engine/.env.example ml-engine/.env
cp frontend/.env.example frontend/.env
# Édite .env avec tes clés API OCR
```

## 🌿 Workflow Git

### Créer une Feature Branch

```bash
# Update develop
git checkout develop
git pull origin develop

# Crée ta branche
git checkout -b feature/nom-descriptif

# Exemples de noms:
# - feature/google-ocr-adapter
# - feature/duplicate-detector
# - fix/amount-validation
# - docs/api-endpoints
```

### Développement

1. **Travaille sur ta feature**
2. **Commits réguliers** (atomiques)
   ```bash
   git add .
   git commit -m "feat(ocr): add Google Document AI adapter with confidence scores"
   ```
3. **Tests locaux**
   ```bash
   # Backend
   pytest
   
   # ML Engine
   pytest
   
   # Frontend
   npm test
   ```

### Pull Request

1. **Push ta branche**
   ```bash
   git push origin feature/nom-descriptif
   ```

2. **Crée PR sur GitHub vers `develop`**
   - Titre clair
   - Description détaillée
   - Screenshots si UI
   - Link vers issue si applicable

3. **Attends review**
   - Réponds aux commentaires
   - Push corrections si demandées

4. **Merge** (après approbation)

## 📝 Standards de Code

### Python

```python
from typing import List, Optional
from decimal import Decimal

async def extract_invoice_data(
    filepath: str,
    ocr_provider: str = "google"
) -> ExtractionResult:
    """
    Extrait données structurées depuis facture PDF/image.
    
    Args:
        filepath: Chemin vers fichier facture
        ocr_provider: Provider OCR (google, aws, paddle)
        
    Returns:
        ExtractionResult avec données + confiance
        
    Raises:
        OCRException: Si extraction échoue
    """
    adapter = get_ocr_adapter(ocr_provider)
    result = await adapter.extract(filepath)
    
    if result.confidence < 0.8:
        logger.warning(f"Low confidence: {result.confidence}")
    
    return result
```

### React

```jsx
import { useState } from 'react';

/**
 * Component for validating invoice suggestions.
 */
export function ValidationCard({ invoice, suggestions, onValidate }) {
  const [selectedAccount, setSelectedAccount] = useState(suggestions[0].account);
  
  const handleValidate = () => {
    onValidate({
      invoiceId: invoice.id,
      validatedAccount: selectedAccount,
      corrected: selectedAccount !== suggestions[0].account
    });
  };
  
  return (
    <div className="border rounded-lg p-4">
      <h3 className="font-bold">{invoice.supplier}</h3>
      
      <div className="space-y-2 mt-4">
        {suggestions.map((s, idx) => (
          <SuggestionOption
            key={s.account}
            suggestion={s}
            rank={idx + 1}
            selected={selectedAccount === s.account}
            onSelect={() => setSelectedAccount(s.account)}
          />
        ))}
      </div>
      
      <button 
        onClick={handleValidate}
        className="mt-4 w-full bg-blue-600 text-white px-4 py-2 rounded"
      >
        Valider
      </button>
    </div>
  );
}
```

## 🧪 Tests

### Écrire des Tests

**Backend (pytest):**
```python
# tests/unit/test_duplicate_detector.py
from decimal import Decimal
from app.core.duplicate_detector import DuplicateDetector

def test_detect_exact_duplicate():
    detector = DuplicateDetector()
    
    # Create existing invoice
    existing = create_test_invoice(
        supplier="SENELEC",
        invoice_number="FACT-2024-001",
        amount=Decimal("150000.00")
    )
    
    # Test with same invoice
    result = detector.detect_duplicates(existing)
    
    assert len(result) == 0  # Should find no duplicates (itself excluded)
```

**Frontend (vitest):**
```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { ValidationCard } from './ValidationCard';

test('validates invoice with suggested account', () => {
  const invoice = { id: '123', supplier: 'SENELEC' };
  const suggestions = [
    { account: '6054', label: 'Électricité', confidence: 0.92 }
  ];
  const onValidate = vi.fn();
  
  render(
    <ValidationCard 
      invoice={invoice}
      suggestions={suggestions}
      onValidate={onValidate}
    />
  );
  
  fireEvent.click(screen.getByText('Valider'));
  
  expect(onValidate).toHaveBeenCalledWith({
    invoiceId: '123',
    validatedAccount: '6054',
    corrected: false
  });
});
```

## 📚 Documentation

- **Code:** Docstrings/comments pour logique complexe
- **API:** Auto-documentée via FastAPI (`/docs`)
- **README:** Update si nouvelles features majeures

## ❓ Questions

- Ouvre une issue sur GitHub
- Tag avec `question` label
- Réponds dans les 24h

## 🎯 Priorités Actuelles

Voir [GitHub Projects](https://github.com/ibrahim123959/AI-INVOICE-AUTOMATION-SENEGAL/projects) pour roadmap.

---

**Merci de contribuer ! 🚀**
EOF
```