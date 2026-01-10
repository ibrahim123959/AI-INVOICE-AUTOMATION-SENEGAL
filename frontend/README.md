# Frontend - Interface Utilisateur Web

## 📖 Contexte

Le frontend est l'interface web que les utilisateurs (comptables, auditeurs) utilisent pour interagir avec la plateforme. Il transforme un processus complexe (upload → OCR → ML → validation → export) en une expérience utilisateur fluide et intuitive.

**Workflow utilisateur typique :**

1. **Upload** : Glisser-déposer factures PDF
2. **Attente** : Voir progression traitement (barre, statut)
3. **Validation** : Voir facture + données extraites côte-à-côte, valider/corriger suggestions
4. **Export** : Télécharger fichier FEC pour Sage
```
┌─────────────────────────────────────────┐
│  Utilisateur (Navigateur Chrome/Firefox)│
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    ► FRONTEND ◄ (Ce module)              │
│                                          │
│  Pages :                                 │
│  - UploadPage : Drag-drop factures       │
│  - ValidationPage : Revue données        │
│  - DashboardPage : Stats, métriques      │
│  - HistoryPage : Historique factures     │
└──────────────┬───────────────────────────┘
               ↓ (Appels API REST)
┌──────────────────────────────────────────┐
│           BACKEND API                    │
│  POST /upload, GET /invoices, etc.       │
└──────────────────────────────────────────┘
```

## 🏗️ Architecture
```
frontend/
│
├── public/                      # Assets statiques
│   ├── index.html               # Point entrée HTML
│   └── favicon.ico              # Icône application
│
├── src/                         # Code source React
│   ├── main.jsx                 # Point entrée JS (ReactDOM.render)
│   ├── App.jsx                  # Composant racine + routing
│   │
│   ├── pages/                   # Pages principales
│   │   ├── UploadPage.jsx       # Page upload factures
│   │   │                        # Drag-drop, sélection fichiers
│   │   │                        # Affiche liste fichiers sélectionnés
│   │   │                        # Bouton "Traiter" lance upload
│   │   ├── ValidationPage.jsx   # Page validation extractions
│   │   │                        # Vue split : PDF gauche, données droite
│   │   │                        # Suggestions comptes avec scores confiance
│   │   │                        # Boutons Valider/Corriger
│   │   ├── DashboardPage.jsx    # Dashboard statistiques
│   │   │                        # Métriques : taux matching, factures/jour
│   │   │                        # Graphiques : évolution, répartition
│   │   └── HistoryPage.jsx      # Historique factures traitées
│   │                            # Tableau filtrable, recherche
│   │                            # Export CSV historique
│   │
│   ├── components/              # Composants réutilisables
│   │   ├── common/              # UI génériques
│   │   │   ├── Button.jsx       # Bouton stylisé Tailwind
│   │   │   ├── Modal.jsx        # Modal overlay
│   │   │   └── LoadingSpinner.jsx # Spinner chargement
│   │   │
│   │   ├── invoice/             # Spécifiques factures
│   │   │   ├── InvoiceUploader.jsx    # Zone drag-drop
│   │   │   ├── InvoicePreview.jsx     # Affichage PDF (iframe/canvas)
│   │   │   ├── ExtractionDisplay.jsx  # Données extraites formatées
│   │   │   └── ImputationSuggestion.jsx # Widget suggestions comptes
│   │   │
│   │   └── validation/          # Workflow validation
│   │       ├── ValidationQueue.jsx    # Liste factures à valider
│   │       ├── SideBySideView.jsx     # PDF + données côte-à-côte
│   │       └── AccountSelector.jsx    # Dropdown comptes SYSCOHADA
│   │
│   ├── services/                # Communication API
│   │   ├── api.js               # Client Axios configuré
│   │   │                        # Base URL, interceptors, gestion erreurs
│   │   ├── invoiceService.js    # Endpoints factures
│   │   │                        # uploadInvoice(), getInvoice(), etc.
│   │   ├── validationService.js # Endpoints validation
│   │   │                        # validateImputation(), correctAccount()
│   │   └── exportService.js     # Endpoints exports
│   │                            # generateFEC(), downloadExport()
│   │
│   ├── hooks/                   # Custom React hooks
│   │   ├── useInvoiceUpload.js  # Logique upload (progress, errors)
│   │   ├── useValidation.js     # Logique validation (state, actions)
│   │   └── useAuth.js           # Authentification (futur)
│   │
│   ├── store/                   # State management (Zustand)
│   │   ├── invoiceStore.js      # État global factures
│   │   │                        # Liste factures, statuts, sélection
│   │   └── uiStore.js           # État UI (modales, notifications)
│   │
│   └── utils/                   # Utilitaires
│       ├── formatters.js        # Formatage dates, montants
│       │                        # 150000 → "150 000 FCFA"
│       └── validators.js        # Validations côté client
│
├── tests/                       # Tests frontend
│   ├── unit/                    # Tests composants isolés (Vitest)
│   └── e2e/                     # Tests end-to-end (Playwright)
│
├── Dockerfile                   # Image Docker production
├── package.json                 # Dépendances Node
├── vite.config.js               # Config build Vite
├── tailwind.config.js           # Config Tailwind CSS
└── .env.example                 # Template vars environnement
```

## 🎨 Stack Technique

- **Framework** : React 18 (Hooks, Functional Components)
- **Build Tool** : Vite (ultra-rapide, HMR instantané)
- **Styling** : Tailwind CSS (utility-first, responsive)
- **State** : Zustand (simple, pas de boilerplate Redux)
- **HTTP** : Axios (interceptors, gestion erreurs)
- **Routing** : React Router v6
- **Icons** : Lucide React (légères, personnalisables)

## 🚀 Démarrage

### Prérequis

- **Node.js 20+** ([Télécharger](https://nodejs.org/))
- **npm 10+** (inclus avec Node)

### Installation
```bash
cd frontend

# Installer dépendances
npm install

# Configurer variables environnement
cp .env.example .env
# Éditer .env :
# VITE_API_URL=http://localhost:8000
```

### Développement
```bash
# Lancer dev server (HMR activé)
npm run dev

# Application disponible : http://localhost:5173
# Changes auto-détectés, page reload instantané
```

### Build Production
```bash
# Créer build optimisé
npm run build

# Dossier dist/ contient assets minifiés, prêts deploy

# Prévisualiser build
npm run preview
```

### Tests
```bash
# Tests unitaires
npm run test

# Tests e2e (nécessite backend running)
npm run test:e2e
```

## 🎯 Pages Principales

### UploadPage

**Fonctionnalités :**
- Zone drag-and-drop (glisser fichiers depuis explorateur)
- Sélection multiple fichiers (Click "Parcourir")
- Prévisualisation fichiers sélectionnés (nom, taille, type)
- Validation côté client (format PDF/JPG/PNG, max 10MB)
- Progress bar upload
- Gestion erreurs (fichier invalide, upload échoué)

**Flow :**
```
1. User drop 5 PDFs → Affiche liste 5 fichiers
2. Click "Traiter" → Upload vers /api/v1/invoices/upload
3. Progress bar 0% → 100%
4. Redirect auto vers ValidationPage
```

### ValidationPage

**Fonctionnalités :**
- Liste factures à valider (queue)
- Vue split-screen :
  - Gauche : PDF facture (iframe ou canvas)
  - Droite : Données extraites + suggestions
- Highlight confiance :
  - Vert (>90%) : Haute confiance, validation rapide
  - Orange (70-90%) : Moyenne, vérifier
  - Rouge (<70%) : Faible, attention
- Boutons actions :
  - ✅ Valider : Accepte suggestion
  - ✏️ Modifier : Change compte
  - ❌ Rejeter : Marque invalide
- Raccourcis clavier (V = valider, M = modifier, N = suivant)

**Flow :**
```
1. Charge première facture queue
2. User voit PDF + suggestion "6054 - Électricité (92%)"
3. Press "V" → Valide, passe suivante automatiquement
4. Suggestion ambiguë → Click "Modifier" → Dropdown comptes → Sélectionne → Sauvegarde
```

### DashboardPage

**Métriques affichées :**
- Total factures traitées (mois en cours)
- Taux matching automatique (85% en moyenne)
- Temps économisé vs saisie manuelle
- Top 5 fournisseurs (volume)
- Répartition par compte comptable
- Évolution temporelle (graphique ligne)

**Graphiques (Recharts) :**
- Line chart : Factures/jour sur 30 jours
- Pie chart : Répartition par statut (validées, en attente, rejetées)
- Bar chart : Top comptes utilisés

### HistoryPage

**Fonctionnalités :**
- Tableau toutes factures (paginé)
- Colonnes : Date, Fournisseur, Montant, Compte, Statut, Actions
- Recherche full-text (nom fournisseur, montant)
- Filtres : Date range, Statut, Compte comptable
- Tri colonnes (click header)
- Actions ligne :
  - 👁️ Voir détails
  - 📄 Télécharger PDF original
  - 📊 Voir écriture comptable
- Export CSV complet historique

## 🔗 Communication API

### Configuration Axios
```javascript
// services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,  // http://localhost:8000
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Interceptor erreurs
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Redirect login (futur)
    }
    if (error.response?.status === 500) {
      // Affiche erreur serveur
      toast.error("Erreur serveur. Veuillez réessayer.");
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Exemple Service
```javascript
// services/invoiceService.js
import api from './api';

export const uploadInvoice = async (file) => {
  const formData = new FormData();
  formData.append('file', file);
  
  const response = await api.post('/v1/invoices/upload', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: (progressEvent) => {
      const progress = (progressEvent.loaded / progressEvent.total) * 100;
      // Update progress bar
    }
  });
  
  return response.data;
};

export const getInvoice = async (invoiceId) => {
  const response = await api.get(`/v1/invoices/${invoiceId}`);
  return response.data;
};

export const validateImputation = async (invoiceId, accountCode) => {
  const response = await api.post(`/v1/validation/${invoiceId}`, {
    account_code: accountCode,
    validated: true
  });
  return response.data;
};
```

## 🎨 Design System (Tailwind)

### Couleurs
```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: '#3B82F6',    // Bleu actions principales
        success: '#10B981',    // Vert validations
        warning: '#F59E0B',    // Orange alertes
        danger: '#EF4444',     // Rouge erreurs
        neutral: '#6B7280'     // Gris textes secondaires
      }
    }
  }
}
```

### Composants Réutilisables

**Button :**
```jsx
<Button variant="primary" size="lg" onClick={handleSubmit}>
  Valider
</Button>

// Variants : primary, secondary, danger
// Sizes : sm, md, lg
```

**Modal :**
```jsx
<Modal isOpen={showModal} onClose={() => setShowModal(false)}>
  <Modal.Header>Confirmer suppression</Modal.Header>
  <Modal.Body>Êtes-vous sûr ?</Modal.Body>
  <Modal.Footer>
    <Button onClick={handleDelete}>Supprimer</Button>
  </Modal.Footer>
</Modal>
```

## 📱 Responsive Design

**Breakpoints Tailwind :**
- `sm:` 640px (tablettes portrait)
- `md:` 768px (tablettes paysage)
- `lg:` 1024px (laptop)
- `xl:` 1280px (desktop)

**Approche :**
- Desktop-first (optimisé pour comptables sur PC)
- Mobile : lecture seule (consultation historique, pas de validation)
- Tablette : workflow validation simplifié

## 🧪 Tests

### Tests Unitaires (Vitest)
```javascript
// tests/unit/Button.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Button from '@/components/common/Button';

test('Button click appelle callback', () => {
  const handleClick = vi.fn();
  render(<Button onClick={handleClick}>Click me</Button>);
  
  fireEvent.click(screen.getByText('Click me'));
  
  expect(handleClick).toHaveBeenCalledTimes(1);
});
```

### Tests E2E (Playwright)
```javascript
// tests/e2e/upload-flow.spec.js
test('Upload et validation facture complète', async ({ page }) => {
  await page.goto('http://localhost:5173');
  
  // Upload facture
  await page.setInputFiles('input[type="file"]', 'tests/fixtures/senelec.pdf');
  await page.click('button:has-text("Traiter")');
  
  // Attend traitement
  await page.waitForSelector('text=Traitement terminé');
  
  // Navigue vers validation
  await page.click('a:has-text("Valider")');
  
  // Vérifie suggestion affichée
  await expect(page.locator('text=6054 - Électricité')).toBeVisible();
  
  // Valide
  await page.click('button:has-text("Valider")');
  
  // Vérifie succès
  await expect(page.locator('text=Facture validée')).toBeVisible();
});
```

## 🔐 Sécurité

**Protection XSS :**
React échappe automatiquement contenu dynamique. Éviter `dangerouslySetInnerHTML`.

**Validation Inputs :**
```javascript
// Valide côté client AVANT envoi API
const validateAmount = (amount) => {
  if (isNaN(amount) || amount <= 0) {
    throw new Error("Montant invalide");
  }
};
```

**HTTPS Only (Production) :**
Toutes requêtes API via HTTPS. Config Vite production force HTTPS.

## 📚 Ressources

- [React Documentation](https://react.dev/)
- [Vite Guide](https://vitejs.dev/guide/)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [Zustand](https://github.com/pmndrs/zustand)
- [React Router](https://reactrouter.com/)
- [Vitest](https://vitest.dev/)
- [Playwright](https://playwright.dev/)

## ❓ FAQ

**Q : Pourquoi Vite et pas Create React App ?**  
R : Vite 10-100x plus rapide (HMR instantané). CRA deprecated, Vite standard moderne.

**Q : Pourquoi Zustand et pas Redux ?**  
R : Zustand ultra-simple (pas de boilerplate), performant, suffit pour notre use case. Redux overkill ici.

**Q : Frontend peut fonctionner sans backend ?**  
R : Mode mock disponible (dev) avec données factices. Prod nécessite backend.

**Q : Accessibilité (a11y) ?**  
R : Composants respectent WCAG 2.1 AA (contraste, navigation clavier, screen readers).

---

*Version : 0.1.0*  
*Framework : React 18 + Vite 5 + Tailwind 3*