# Implementation Specifications

## 🔧 Technical Implementation Details

### 1. Security Implementation

#### **API Proxy Server (Vercel/Netlify)**

**File Structure:**
```
/api/
├── generate-vocab.js
├── chat.js
└── examples.js
```

**Implementation:**
```javascript
// api/generate-vocab.js
export default async function handler(req, res) {
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGINS);
  res.setHeader('Access-Control-Allow-Methods', 'POST');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
  
  if (req.method === 'OPTIONS') {
    res.status(200).end();
    return;
  }
  
  if (req.method !== 'POST') {
    res.status(405).json({ error: 'Method not allowed' });
    return;
  }
  
  // Rate limiting
  const clientIP = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  if (await isRateLimited(clientIP)) {
    res.status(429).json({ error: 'Rate limit exceeded' });
    return;
  }
  
  // Input validation
  const { prompt, difficulty, language } = req.body;
  if (!prompt || !difficulty || !language) {
    res.status(400).json({ error: 'Missing required fields' });
    return;
  }
  
  try {
    const response = await fetch('https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.GEMINI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        contents: [{ role: 'user', parts: [{ text: prompt }] }],
        generationConfig: {
          temperature: 0.7,
          topP: 0.8,
          topK: 40,
          maxOutputTokens: 2048
        }
      })
    });
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }
    
    const data = await response.json();
    res.status(200).json(data);
    
  } catch (error) {
    console.error('API Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
}

// Rate limiting implementation
const rateLimitMap = new Map();
async function isRateLimited(clientIP) {
  const now = Date.now();
  const windowMs = 15 * 60 * 1000; // 15 minutes
  const maxRequests = 100;
  
  const clientData = rateLimitMap.get(clientIP) || { count: 0, resetTime: now + windowMs };
  
  if (now > clientData.resetTime) {
    clientData.count = 1;
    clientData.resetTime = now + windowMs;
  } else {
    clientData.count++;
  }
  
  rateLimitMap.set(clientIP, clientData);
  return clientData.count > maxRequests;
}
```

#### **Client-Side Security Updates**

```javascript
// js/api-service.js
class APIService {
  constructor() {
    this.baseURL = process.env.NODE_ENV === 'production' 
      ? 'https://your-app.vercel.app/api'
      : 'http://localhost:3000/api';
    this.requestQueue = [];
    this.isOnline = navigator.onLine;
    
    // Listen for online/offline events
    window.addEventListener('online', () => this.handleOnline());
    window.addEventListener('offline', () => this.handleOffline());
  }
  
  async generateVocabulary(difficulty, language, count = 10) {
    const prompt = this.createPrompt(difficulty, language, count);
    
    try {
      const response = await this.makeRequest('/generate-vocab', {
        prompt,
        difficulty,
        language,
        count
      });
      
      return this.parseVocabularyResponse(response);
    } catch (error) {
      if (!this.isOnline) {
        // Queue request for when online
        this.queueRequest('generateVocabulary', arguments);
        throw new Error('Offline: Request queued for when connection is restored');
      }
      throw error;
    }
  }
  
  async makeRequest(endpoint, data) {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data)
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'API request failed');
    }
    
    return response.json();
  }
  
  queueRequest(method, args) {
    this.requestQueue.push({ method, args, timestamp: Date.now() });
  }
  
  async handleOnline() {
    this.isOnline = true;
    
    // Process queued requests
    while (this.requestQueue.length > 0) {
      const { method, args } = this.requestQueue.shift();
      try {
        await this[method](...args);
      } catch (error) {
        console.error('Failed to process queued request:', error);
      }
    }
  }
  
  handleOffline() {
    this.isOnline = false;
  }
}
```

### 2. Modular Architecture Implementation

#### **Module System Setup**

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>AI Vocabulary Cards</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="manifest" href="manifest.json">
  <link rel="stylesheet" href="css/main.css">
</head>
<body>
  <div id="app"></div>
  
  <!-- Load modules -->
  <script type="module" src="js/app.js"></script>
</body>
</html>
```

```javascript
// js/app.js - Main application entry point
import { APIService } from './services/api-service.js';
import { StorageService } from './services/storage-service.js';
import { TTSService } from './services/tts-service.js';
import { UIManager } from './ui/ui-manager.js';
import { StateManager } from './state/state-manager.js';
import { Router } from './router/router.js';

class VocabApp {
  constructor() {
    this.services = {
      api: new APIService(),
      storage: new StorageService(),
      tts: new TTSService()
    };
    
    this.state = new StateManager();
    this.ui = new UIManager(this.services, this.state);
    this.router = new Router(this.ui);
  }
  
  async init() {
    try {
      // Initialize services
      await this.initializeServices();
      
      // Load user settings
      await this.loadUserSettings();
      
      // Initialize UI
      await this.ui.init();
      
      // Start router
      this.router.start();
      
      // Register service worker
      await this.registerServiceWorker();
      
      console.log('App initialized successfully');
    } catch (error) {
      console.error('Failed to initialize app:', error);
      this.handleInitializationError(error);
    }
  }
  
  async initializeServices() {
    await Promise.all([
      this.services.storage.init(),
      this.services.tts.init(),
      this.services.api.init()
    ]);
  }
  
  async loadUserSettings() {
    const settings = await this.services.storage.getSettings();
    this.state.setState({ settings });
  }
  
  async registerServiceWorker() {
    if ('serviceWorker' in navigator) {
      try {
        const registration = await navigator.serviceWorker.register('/sw.js');
        console.log('Service Worker registered:', registration);
      } catch (error) {
        console.error('Service Worker registration failed:', error);
      }
    }
  }
  
  handleInitializationError(error) {
    document.getElementById('app').innerHTML = `
      <div class="error-container">
        <h1>Failed to load application</h1>
        <p>${error.message}</p>
        <button onclick="location.reload()">Retry</button>
      </div>
    `;
  }
}

// Initialize app when DOM is ready
document.addEventListener('DOMContentLoaded', () => {
  const app = new VocabApp();
  app.init();
});
```

#### **State Management System**

```javascript
// state/state-manager.js
export class StateManager {
  constructor() {
    this.state = {
      vocabulary: [],
      settings: {
        theme: 'auto',
        language: { from: 'en', to: 'zh' },
        difficulty: 'medium',
        tts: {
          englishVoice: null,
          chineseVoice: null,
          rate: 1.0,
          pitch: 1.0
        }
      },
      ui: {
        currentView: 'home',
        isLoading: false,
        searchQuery: '',
        filters: {}
      },
      user: {
        progress: {},
        achievements: [],
        streak: 0
      }
    };
    
    this.listeners = new Map();
    this.middleware = [];
  }
  
  // Subscribe to state changes
  subscribe(path, callback) {
    if (!this.listeners.has(path)) {
      this.listeners.set(path, new Set());
    }
    this.listeners.get(path).add(callback);
    
    // Return unsubscribe function
    return () => {
      const callbacks = this.listeners.get(path);
      if (callbacks) {
        callbacks.delete(callback);
      }
    };
  }
  
  // Update state
  setState(updates, path = '') {
    const oldState = this.getState(path);
    
    if (path) {
      this.setNestedState(path, updates);
    } else {
      this.state = { ...this.state, ...updates };
    }
    
    // Run middleware
    this.runMiddleware(oldState, this.getState(path), path);
    
    // Notify listeners
    this.notifyListeners(path);
  }
  
  // Get state
  getState(path = '') {
    if (!path) return this.state;
    
    return path.split('.').reduce((obj, key) => obj?.[key], this.state);
  }
  
  // Add middleware
  addMiddleware(middleware) {
    this.middleware.push(middleware);
  }
  
  setNestedState(path, value) {
    const keys = path.split('.');
    const lastKey = keys.pop();
    const target = keys.reduce((obj, key) => obj[key], this.state);
    target[lastKey] = { ...target[lastKey], ...value };
  }
  
  runMiddleware(oldState, newState, path) {
    this.middleware.forEach(middleware => {
      middleware(oldState, newState, path);
    });
  }
  
  notifyListeners(path) {
    // Notify specific path listeners
    const callbacks = this.listeners.get(path);
    if (callbacks) {
      callbacks.forEach(callback => callback(this.getState(path)));
    }
    
    // Notify global listeners
    const globalCallbacks = this.listeners.get('');
    if (globalCallbacks) {
      globalCallbacks.forEach(callback => callback(this.state));
    }
  }
}
```

#### **Component System**

```javascript
// components/base-component.js
export class BaseComponent {
  constructor(props = {}) {
    this.props = props;
    this.element = null;
    this.children = [];
    this.eventListeners = [];
  }
  
  render() {
    if (this.element) {
      this.cleanup();
    }
    
    this.element = this.createElement();
    this.bindEvents();
    this.renderChildren();
    
    return this.element;
  }
  
  createElement() {
    // Override in subclasses
    return document.createElement('div');
  }
  
  bindEvents() {
    // Override in subclasses
  }
  
  renderChildren() {
    this.children.forEach(child => {
      if (child.render) {
        this.element.appendChild(child.render());
      }
    });
  }
  
  addEventListener(element, event, handler) {
    element.addEventListener(event, handler);
    this.eventListeners.push({ element, event, handler });
  }
  
  cleanup() {
    // Remove event listeners
    this.eventListeners.forEach(({ element, event, handler }) => {
      element.removeEventListener(event, handler);
    });
    this.eventListeners = [];
    
    // Cleanup children
    this.children.forEach(child => {
      if (child.cleanup) {
        child.cleanup();
      }
    });
  }
  
  updateProps(newProps) {
    this.props = { ...this.props, ...newProps };
    this.render();
  }
}
```

```javascript
// components/vocab-card.js
import { BaseComponent } from './base-component.js';

export class VocabCard extends BaseComponent {
  constructor(props) {
    super(props);
    this.word = props.word;
    this.onSave = props.onSave;
    this.onSpeak = props.onSpeak;
    this.onClick = props.onClick;
  }
  
  createElement() {
    const card = document.createElement('div');
    card.className = 'vocab-card';
    card.setAttribute('role', 'button');
    card.setAttribute('tabindex', '0');
    card.setAttribute('aria-label', 
      `Vocabulary card: ${this.word.en} means ${this.word.zh}. Press Enter to view details.`
    );
    
    card.innerHTML = `
      <div class="card-header">
        <span class="word-primary">${this.word.en}</span>
        <button class="speak-btn" data-lang="en" aria-label="Pronounce ${this.word.en}">
          🔊
        </button>
      </div>
      <div class="word-secondary">${this.word.zh}</div>
      <div class="card-example">
        <strong>Example:</strong> ${this.word.en_example}
        <button class="speak-btn" data-lang="en" aria-label="Read example">🔊</button>
      </div>
      <div class="card-actions">
        <button class="save-btn" aria-label="Save word">⭐</button>
        <span class="difficulty-indicator" title="Difficulty: ${this.props.difficulty}">
          ${this.getDifficultyIcon()}
        </span>
      </div>
    `;
    
    return card;
  }
  
  bindEvents() {
    // Main card click
    this.addEventListener(this.element, 'click', (e) => {
      if (!e.target.closest('button')) {
        this.onClick?.(this.word);
      }
    });
    
    // Keyboard navigation
    this.addEventListener(this.element, 'keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.onClick?.(this.word);
      }
    });
    
    // Speak buttons
    this.element.querySelectorAll('.speak-btn').forEach(btn => {
      this.addEventListener(btn, 'click', (e) => {
        e.stopPropagation();
        const lang = btn.dataset.lang;
        const text = lang === 'en' ? this.word.en : this.word.zh;
        this.onSpeak?.(text, lang);
      });
    });
    
    // Save button
    const saveBtn = this.element.querySelector('.save-btn');
    this.addEventListener(saveBtn, 'click', (e) => {
      e.stopPropagation();
      this.onSave?.(this.word);
    });
  }
  
  getDifficultyIcon() {
    const icons = {
      easy: '🟢',
      medium: '🟠',
      hard: '🔴'
    };
    return icons[this.props.difficulty] || '⚪';
  }
}
```

### 3. Testing Strategy

#### **Unit Testing Setup**

```javascript
// tests/setup.js
import { jest } from '@jest/globals';

// Mock DOM environment
global.document = {
  createElement: jest.fn(),
  addEventListener: jest.fn(),
  querySelector: jest.fn(),
  querySelectorAll: jest.fn()
};

global.window = {
  localStorage: {
    getItem: jest.fn(),
    setItem: jest.fn(),
    removeItem: jest.fn()
  },
  addEventListener: jest.fn()
};

global.navigator = {
  serviceWorker: {
    register: jest.fn()
  }
};
```

```javascript
// tests/components/vocab-card.test.js
import { VocabCard } from '../../components/vocab-card.js';

describe('VocabCard', () => {
  const mockWord = {
    en: 'apple',
    zh: '苹果',
    en_example: 'I eat an apple.',
    zh_example: '我吃苹果。'
  };
  
  let card;
  let mockProps;
  
  beforeEach(() => {
    mockProps = {
      word: mockWord,
      difficulty: 'easy',
      onSave: jest.fn(),
      onSpeak: jest.fn(),
      onClick: jest.fn()
    };
    
    card = new VocabCard(mockProps);
  });
  
  test('renders word correctly', () => {
    const element = card.render();
    
    expect(element.querySelector('.word-primary').textContent).toBe('apple');
    expect(element.querySelector('.word-secondary').textContent).toBe('苹果');
  });
  
  test('calls onSpeak when speak button clicked', () => {
    const element = card.render();
    const speakBtn = element.querySelector('.speak-btn');
    
    speakBtn.click();
    
    expect(mockProps.onSpeak).toHaveBeenCalledWith('apple', 'en');
  });
  
  test('calls onClick when card clicked', () => {
    const element = card.render();
    
    element.click();
    
    expect(mockProps.onClick).toHaveBeenCalledWith(mockWord);
  });
});
```

#### **Integration Testing**

```javascript
// tests/integration/api-service.test.js
import { APIService } from '../../js/services/api-service.js';

describe('APIService Integration', () => {
  let apiService;
  
  beforeEach(() => {
    apiService = new APIService();
  });
  
  test('generates vocabulary successfully', async () => {
    const result = await apiService.generateVocabulary('easy', 'en-zh', 5);
    
    expect(result).toHaveLength(5);
    expect(result[0]).toHaveProperty('en');
    expect(result[0]).toHaveProperty('zh');
    expect(result[0]).toHaveProperty('en_example');
    expect(result[0]).toHaveProperty('zh_example');
  });
  
  test('handles API errors gracefully', async () => {
    // Mock network error
    global.fetch = jest.fn().mockRejectedValue(new Error('Network error'));
    
    await expect(apiService.generateVocabulary('easy', 'en-zh', 5))
      .rejects.toThrow('Network error');
  });
});
```

#### **E2E Testing with Playwright**

```javascript
// tests/e2e/vocabulary-learning.spec.js
import { test, expect } from '@playwright/test';

test.describe('Vocabulary Learning Flow', () => {
  test('user can learn vocabulary cards', async ({ page }) => {
    await page.goto('/');
    
    // Wait for cards to load
    await page.waitForSelector('.vocab-card');
    
    // Check that cards are displayed
    const cards = await page.locator('.vocab-card').count();
    expect(cards).toBeGreaterThan(0);
    
    // Click on first card
    await page.locator('.vocab-card').first().click();
    
    // Modal should open
    await expect(page.locator('.modal')).toBeVisible();
    
    // Test TTS functionality
    await page.locator('.speak-btn').first().click();
    
    // Save word
    await page.locator('.save-btn').click();
    
    // Check toast notification
    await expect(page.locator('.toast')).toContainText('Word saved!');
  });
  
  test('user can change settings', async ({ page }) => {
    await page.goto('/');
    
    // Open settings
    await page.locator('#settingsBtn').click();
    
    // Change difficulty
    await page.selectOption('#levelSelect', 'hard');
    
    // Save settings
    await page.locator('#saveBtn').click();
    
    // Verify settings persisted
    await page.reload();
    await page.locator('#settingsBtn').click();
    
    const selectedValue = await page.locator('#levelSelect').inputValue();
    expect(selectedValue).toBe('hard');
  });
});
```

### 4. Migration Strategy

#### **Backward Compatibility**

```javascript
// js/migration-manager.js
export class MigrationManager {
  constructor(storageService) {
    this.storage = storageService;
    this.currentVersion = '2.0.0';
    this.migrations = [
      { version: '1.1.0', migrate: this.migrateToV1_1 },
      { version: '2.0.0', migrate: this.migrateToV2_0 }
    ];
  }
  
  async runMigrations() {
    const currentVersion = await this.storage.getVersion() || '1.0.0';
    
    for (const migration of this.migrations) {
      if (this.shouldRunMigration(currentVersion, migration.version)) {
        console.log(`Running migration to ${migration.version}`);
        await migration.migrate();
        await this.storage.setVersion(migration.version);
      }
    }
  }
  
  shouldRunMigration(current, target) {
    return this.compareVersions(current, target) < 0;
  }
  
  compareVersions(a, b) {
    const aParts = a.split('.').map(Number);
    const bParts = b.split('.').map(Number);
    
    for (let i = 0; i < Math.max(aParts.length, bParts.length); i++) {
      const aPart = aParts[i] || 0;
      const bPart = bParts[i] || 0;
      
      if (aPart < bPart) return -1;
      if (aPart > bPart) return 1;
    }
    
    return 0;
  }
  
  async migrateToV1_1() {
    // Migrate old settings format
    const oldSettings = await this.storage.get('settings');
    if (oldSettings && !oldSettings.tts) {
      const newSettings = {
        ...oldSettings,
        tts: {
          englishVoice: oldSettings.selectedVoice || null,
          chineseVoice: null,
          rate: 1.0,
          pitch: 1.0
        }
      };
      await this.storage.set('settings', newSettings);
    }
  }
  
  async migrateToV2_0() {
    // Migrate vocabulary data structure
    const oldVocab = await this.storage.get('vocabulary');
    if (oldVocab && Array.isArray(oldVocab)) {
      const newVocab = oldVocab.map(word => ({
        ...word,
        id: this.generateId(),
        createdAt: Date.now(),
        progress: {
          streak: 0,
          lastReview: null,
          nextReview: null,
          difficulty: 1.0
        }
      }));
      await this.storage.set('vocabulary', newVocab);
    }
  }
  
  generateId() {
    return Math.random().toString(36).substr(2, 9);
  }
}
```

This comprehensive implementation plan provides:

1. **Security-first approach** with API proxy and input sanitization
2. **Modular architecture** for better maintainability
3. **Component-based UI** system for reusability
4. **Comprehensive testing** strategy covering unit, integration, and E2E tests
5. **Migration system** for backward compatibility
6. **Performance optimizations** with caching and memory management

The implementation follows modern web development best practices while maintaining the simplicity and effectiveness of the current application.
