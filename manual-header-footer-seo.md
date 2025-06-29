# Manual Técnico: Sistema Unificado Header/Footer con Optimización SEO
## ArcaTierra - Versión 1.0

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [Arquitectura del Sistema](#arquitectura-del-sistema)
3. [Implementación del Header](#implementación-del-header)
4. [Implementación del Footer](#implementación-del-footer)
5. [Sistema de Gestión de Estado](#sistema-de-gestión-de-estado)
6. [Optimización SEO](#optimización-seo)
7. [Responsive Design](#responsive-design)
8. [Testing y Validación](#testing-y-validación)
9. [Checklist de Implementación](#checklist-de-implementación)
10. [Troubleshooting](#troubleshooting)

---

## 1. Resumen Ejecutivo

### Objetivo
Implementar un sistema de navegación unificado (header/footer) que mantenga consistencia visual, funcionalidad dinámica basada en estado de autenticación, y cumpla con las mejores prácticas SEO para mejorar el posicionamiento orgánico de ArcaTierra.

### Características Principales
- **Navegación Consistente**: Mismo header/footer en todas las páginas
- **Estado Dinámico**: Adaptación automática según autenticación del usuario
- **SEO Optimizado**: Estructura semántica y metadatos correctos
- **Performance**: Carga rápida y eficiente
- **Responsive**: Adaptación perfecta a todos los dispositivos

### Stack Tecnológico
- HTML5 semántico
- CSS3 con variables personalizadas
- JavaScript vanilla (sin dependencias)
- localStorage para persistencia
- Schema.org para datos estructurados

---

## 2. Arquitectura del Sistema

### Diagrama de Flujo
```
┌─────────────────┐
│   Página Carga  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Verificar Auth  │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌────────┐
│ Login │ │ Logout │
└───┬───┘ └───┬────┘
    │         │
    └────┬────┘
         │
         ▼
┌─────────────────┐
│ Actualizar UI   │
└─────────────────┘
```

### Estructura de Archivos
```
proyecto-arcatierra/
├── shared/
│   ├── header.js
│   ├── footer.js
│   └── auth-manager.js
├── styles/
│   ├── variables.css
│   ├── header.css
│   └── footer.css
├── images/
│   ├── logo-arcatierra.png
│   └── logo-arcatierra-white.png
└── [páginas HTML]
```

---

## 3. Implementación del Header

### 3.1 Estructura HTML Semántica

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <!-- Meta tags SEO críticos -->
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="[Descripción única por página - máximo 155 caracteres]">
    <meta name="keywords" content="productos orgánicos, chinampas, xochimilco, sustentable">
    
    <!-- Open Graph para redes sociales -->
    <meta property="og:title" content="[Título de la página] - ArcaTierra">
    <meta property="og:description" content="[Descripción para compartir]">
    <meta property="og:image" content="https://arcatierra.com/images/og-image.jpg">
    <meta property="og:url" content="https://arcatierra.com/[página-actual]">
    <meta property="og:type" content="website">
    
    <!-- Twitter Card -->
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="[Título] - ArcaTierra">
    <meta name="twitter:description" content="[Descripción]">
    <meta name="twitter:image" content="https://arcatierra.com/images/twitter-card.jpg">
    
    <!-- Canonical URL -->
    <link rel="canonical" href="https://arcatierra.com/[página-actual]">
    
    <title>[Título único de página] - ArcaTierra | Productos Orgánicos de Xochimilco</title>
    
    <!-- Preload recursos críticos -->
    <link rel="preload" href="/images/logo-arcatierra.png" as="image">
    <link rel="preload" href="/fonts/primary-font.woff2" as="font" type="font/woff2" crossorigin>
    
    <!-- CSS Variables -->
    <style>
        :root {
            /* TERRACOTA (40% - Elementos de Acción) */
            --arcatierra-terracota-principal: #B15543;
            --arcatierra-terracota-medio: #BA6440;
            --arcatierra-terracota-oscuro: #975543;
            
            /* VERDE (30% - Información/Educación) */
            --arcatierra-verde-tipografia: #3A4741;
            --arcatierra-verde-principal: #33503E;
            --arcatierra-verde-claro: #475A52;
            
            /* NEUTROS (30% - Fondos/Base) */
            --arcatierra-crema-principal: #E3DBCB;
            --arcatierra-beige-medio: #CCBB9A;
        }
    </style>
</head>
<body>
    <!-- Header semántico con navegación principal -->
    <header class="main-header" role="banner">
        <nav class="main-nav" role="navigation" aria-label="Navegación principal">
            <!-- Logo con alt text descriptivo -->
            <div class="logo-container">
                <a href="/" aria-label="ArcaTierra - Página de inicio">
                    <img src="/images/logo-arcatierra.png" 
                         alt="ArcaTierra - Productos orgánicos de las chinampas de Xochimilco" 
                         width="200" 
                         height="60"
                         loading="eager">
                </a>
            </div>
            
            <!-- Navegación principal con estructura de lista -->
            <ul class="nav-links" role="menubar">
                <li role="none">
                    <a href="/inicio" role="menuitem" 
                       class="nav-link" 
                       aria-current="page">Inicio</a>
                </li>
                <li role="none">
                    <a href="/productos" role="menuitem" 
                       class="nav-link">Productos</a>
                </li>
                <li role="none">
                    <a href="/experiencias" role="menuitem" 
                       class="nav-link">Experiencias</a>
                </li>
                <li role="none">
                    <a href="/nosotros" role="menuitem" 
                       class="nav-link">Nosotros</a>
                </li>
                <li role="none">
                    <a href="/contacto" role="menuitem" 
                       class="nav-link">Contacto</a>
                </li>
            </ul>
            
            <!-- Sección de usuario dinámica -->
            <div id="userSection" class="user-section" aria-live="polite">
                <!-- Contenido dinámico inyectado por JavaScript -->
            </div>
            
            <!-- Botón móvil accesible -->
            <button class="mobile-menu-toggle" 
                    aria-label="Abrir menú de navegación"
                    aria-expanded="false"
                    aria-controls="mobile-nav">
                <span class="hamburger"></span>
            </button>
        </nav>
    </header>
</body>
</html>
```

### 3.2 CSS Optimizado para Header

```css
/* Header Styles - Mobile First */
.main-header {
    background-color: #ffffff;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    position: sticky;
    top: 0;
    z-index: 1000;
    /* Mejora performance en móviles */
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
}

.main-nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 5%;
    max-width: 1400px;
    margin: 0 auto;
}

/* Logo optimización */
.logo-container img {
    height: 60px;
    width: auto;
    /* Prevenir layout shift */
    aspect-ratio: 200/60;
}

/* Enlaces de navegación */
.nav-links {
    display: none;
    list-style: none;
    gap: 2rem;
    margin: 0;
    padding: 0;
}

.nav-link {
    color: var(--arcatierra-verde-tipografia);
    text-decoration: none;
    font-weight: 500;
    padding: 0.5rem 0;
    position: relative;
    transition: color 0.3s ease;
}

/* Estado hover y activo */
.nav-link:hover,
.nav-link[aria-current="page"] {
    color: var(--arcatierra-terracota-principal);
}

.nav-link[aria-current="page"]::after {
    content: '';
    position: absolute;
    bottom: -2px;
    left: 0;
    width: 100%;
    height: 2px;
    background-color: var(--arcatierra-terracota-principal);
}

/* Responsive - Tablet y Desktop */
@media (min-width: 768px) {
    .nav-links {
        display: flex;
    }
    
    .mobile-menu-toggle {
        display: none;
    }
}

/* Optimización para print */
@media print {
    .main-header {
        position: static;
        box-shadow: none;
    }
}
```

### 3.3 JavaScript para Gestión Dinámica

```javascript
// auth-manager.js - Sistema de autenticación
class AuthManager {
    constructor() {
        this.storageKey = 'arcaTierraCurrentUser';
        this.init();
    }
    
    init() {
        // Verificar estado al cargar
        this.checkAuthStatus();
        
        // Escuchar cambios en otras pestañas
        window.addEventListener('storage', (e) => {
            if (e.key === this.storageKey) {
                this.checkAuthStatus();
            }
        });
    }
    
    checkAuthStatus() {
        const userData = this.getCurrentUser();
        this.updateUIBasedOnAuth(userData);
    }
    
    getCurrentUser() {
        try {
            const data = localStorage.getItem(this.storageKey);
            return data ? JSON.parse(data) : null;
        } catch (error) {
            console.error('Error parsing user data:', error);
            return null;
        }
    }
    
    updateUIBasedOnAuth(userData) {
        const userSection = document.getElementById('userSection');
        if (!userSection) return;
        
        if (userData) {
            // Usuario autenticado
            userSection.innerHTML = `
                <div class="user-menu">
                    <button class="user-avatar" 
                            aria-label="Menú de usuario" 
                            aria-expanded="false"
                            onclick="toggleUserMenu()">
                        ${this.getInitials(userData.name)}
                    </button>
                    <div class="user-dropdown" role="menu" hidden>
                        <a href="/dashboard" role="menuitem">Mi Cuenta</a>
                        <a href="/ordenes" role="menuitem">Mis Órdenes</a>
                        <a href="/suscripciones" role="menuitem">Suscripciones</a>
                        <hr>
                        <button onclick="logout()" role="menuitem">Cerrar Sesión</button>
                    </div>
                </div>
            `;
        } else {
            // Usuario no autenticado
            userSection.innerHTML = `
                <a href="/login" class="login-btn">Iniciar Sesión</a>
            `;
        }
    }
    
    getInitials(name) {
        return name
            .split(' ')
            .map(word => word[0])
            .join('')
            .toUpperCase()
            .slice(0, 2);
    }
    
    login(userData) {
        localStorage.setItem(this.storageKey, JSON.stringify(userData));
        this.checkAuthStatus();
        // Evento personalizado para notificar cambios
        window.dispatchEvent(new CustomEvent('authStateChanged', { 
            detail: { isAuthenticated: true, user: userData } 
        }));
    }
    
    logout() {
        localStorage.removeItem(this.storageKey);
        this.checkAuthStatus();
        window.dispatchEvent(new CustomEvent('authStateChanged', { 
            detail: { isAuthenticated: false } 
        }));
        // Redirigir a inicio
        window.location.href = '/';
    }
}

// Inicializar al cargar DOM
document.addEventListener('DOMContentLoaded', () => {
    window.authManager = new AuthManager();
});
```

---

## 4. Implementación del Footer

### 4.1 Estructura HTML del Footer

```html
<!-- Footer semántico con microdatos -->
<footer class="main-footer" role="contentinfo">
    <div class="footer-content">
        <!-- Sección de empresa con Schema.org -->
        <div class="footer-section" itemscope itemtype="https://schema.org/Organization">
            <img src="/images/logo-arcatierra-white.png" 
                 alt="ArcaTierra" 
                 width="180" 
                 height="54"
                 itemprop="logo">
            <p itemprop="description">
                Conectando las chinampas de Xochimilco con tu mesa. 
                Productos orgánicos, frescos y sustentables.
            </p>
            <meta itemprop="name" content="ArcaTierra">
            <meta itemprop="url" content="https://arcatierra.com">
        </div>
        
        <!-- Enlaces rápidos -->
        <div class="footer-section">
            <h3>Enlaces Rápidos</h3>
            <nav aria-label="Enlaces del footer">
                <ul>
                    <li><a href="/productos">Productos Orgánicos</a></li>
                    <li><a href="/experiencias">Experiencias Culinarias</a></li>
                    <li><a href="/productores">Nuestros Productores</a></li>
                    <li><a href="/sustentabilidad">Sustentabilidad</a></li>
                </ul>
            </nav>
        </div>
        
        <!-- Información de contacto -->
        <div class="footer-section" itemscope itemtype="https://schema.org/ContactPoint">
            <h3>Contacto</h3>
            <address>
                <p itemprop="email">📧 info@arcatierra.com</p>
                <p itemprop="telephone">📱 +52 55 1234 5678</p>
                <p itemprop="address">📍 Xochimilco, CDMX</p>
            </address>
            <meta itemprop="contactType" content="customer service">
            <meta itemprop="availableLanguage" content="es-MX">
        </div>
        
        <!-- Redes sociales -->
        <div class="footer-section">
            <h3>Síguenos</h3>
            <div class="social-links">
                <a href="https://facebook.com/arcatierra" 
                   aria-label="Facebook de ArcaTierra"
                   rel="noopener noreferrer" 
                   target="_blank">
                    <svg><!-- Icono Facebook --></svg>
                </a>
                <a href="https://instagram.com/arcatierra" 
                   aria-label="Instagram de ArcaTierra"
                   rel="noopener noreferrer" 
                   target="_blank">
                    <svg><!-- Icono Instagram --></svg>
                </a>
            </div>
        </div>
    </div>
    
    <!-- Footer bottom con información legal -->
    <div class="footer-bottom">
        <p>&copy; 2025 ArcaTierra. Todos los derechos reservados.</p>
        <nav class="legal-links" aria-label="Enlaces legales">
            <a href="/privacidad">Política de Privacidad</a>
            <a href="/terminos">Términos y Condiciones</a>
            <a href="/cookies">Política de Cookies</a>
        </nav>
    </div>
</footer>
```

### 4.2 CSS del Footer

```css
/* Footer Styles */
.main-footer {
    background-color: var(--arcatierra-terracota-principal);
    color: #ffffff;
    padding: 3rem 5% 1rem;
    margin-top: 4rem;
}

.footer-content {
    max-width: 1400px;
    margin: 0 auto;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 2rem;
    margin-bottom: 2rem;
}

.footer-section h3 {
    margin-bottom: 1rem;
    font-size: 1.25rem;
    font-weight: 600;
}

.footer-section ul {
    list-style: none;
    padding: 0;
}

.footer-section a {
    color: #ffffff;
    text-decoration: none;
    opacity: 0.9;
    transition: opacity 0.3s ease;
    display: inline-block;
    padding: 0.25rem 0;
}

.footer-section a:hover,
.footer-section a:focus {
    opacity: 1;
    text-decoration: underline;
}

/* Footer bottom */
.footer-bottom {
    border-top: 1px solid rgba(255, 255, 255, 0.2);
    padding-top: 1.5rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    gap: 1rem;
}

.legal-links {
    display: flex;
    gap: 1.5rem;
}

.legal-links a {
    color: #ffffff;
    opacity: 0.8;
    text-decoration: none;
    font-size: 0.875rem;
}

/* Responsive Footer */
@media (max-width: 768px) {
    .footer-content {
        grid-template-columns: 1fr;
        text-align: center;
    }
    
    .footer-bottom {
        flex-direction: column;
        text-align: center;
    }
    
    .legal-links {
        flex-direction: column;
        gap: 0.5rem;
    }
}
```

---

## 5. Sistema de Gestión de Estado

### 5.1 LocalStorage Keys y Estructura

```javascript
// storage-config.js
const STORAGE_KEYS = {
    USER: 'arcaTierraCurrentUser',
    CART: 'arcaTierraCart',
    BOOKINGS: 'arcaTierraBookings',
    METRICS: 'arcaTierraMetrics',
    PREFERENCES: 'arcaTierraPreferences'
};

// Estructura de datos
const dataStructures = {
    user: {
        id: 'string',
        name: 'string',
        email: 'string',
        image: 'string|null',
        role: 'customer|admin',
        createdAt: 'ISO 8601 date'
    },
    cart: {
        items: [
            {
                productId: 'string',
                name: 'string',
                price: 'number',
                quantity: 'number',
                image: 'string'
            }
        ],
        total: 'number',
        updatedAt: 'ISO 8601 date'
    },
    preferences: {
        theme: 'light|dark',
        newsletter: 'boolean',
        language: 'es|en'
    }
};
```

### 5.2 State Manager Completo

```javascript
// state-manager.js
class StateManager {
    constructor() {
        this.subscribers = new Map();
        this.state = this.loadState();
    }
    
    loadState() {
        const state = {};
        Object.entries(STORAGE_KEYS).forEach(([key, storageKey]) => {
            try {
                const data = localStorage.getItem(storageKey);
                state[key.toLowerCase()] = data ? JSON.parse(data) : null;
            } catch (error) {
                console.error(`Error loading ${key}:`, error);
                state[key.toLowerCase()] = null;
            }
        });
        return state;
    }
    
    get(key) {
        return this.state[key];
    }
    
    set(key, value) {
        this.state[key] = value;
        const storageKey = STORAGE_KEYS[key.toUpperCase()];
        
        if (value === null) {
            localStorage.removeItem(storageKey);
        } else {
            localStorage.setItem(storageKey, JSON.stringify(value));
        }
        
        this.notify(key, value);
    }
    
    subscribe(key, callback) {
        if (!this.subscribers.has(key)) {
            this.subscribers.set(key, new Set());
        }
        this.subscribers.get(key).add(callback);
        
        // Retornar función para desuscribirse
        return () => {
            this.subscribers.get(key)?.delete(callback);
        };
    }
    
    notify(key, value) {
        this.subscribers.get(key)?.forEach(callback => {
            callback(value);
        });
    }
}

// Instancia global
window.stateManager = new StateManager();
```

---

## 6. Optimización SEO

### 6.1 Metadatos Dinámicos

```javascript
// seo-manager.js
class SEOManager {
    constructor() {
        this.defaultMeta = {
            title: 'ArcaTierra | Productos Orgánicos de Xochimilco',
            description: 'Descubre productos orgánicos frescos de las chinampas de Xochimilco. Entrega a domicilio en CDMX.',
            keywords: 'productos orgánicos, chinampas, xochimilco, sustentable, cdmx',
            image: 'https://arcatierra.com/images/og-default.jpg'
        };
    }
    
    updateMetaTags(pageData) {
        // Title
        document.title = pageData.title || this.defaultMeta.title;
        
        // Description
        this.updateMetaTag('description', pageData.description || this.defaultMeta.description);
        
        // Keywords
        this.updateMetaTag('keywords', pageData.keywords || this.defaultMeta.keywords);
        
        // Open Graph
        this.updateMetaProperty('og:title', pageData.title || this.defaultMeta.title);
        this.updateMetaProperty('og:description', pageData.description || this.defaultMeta.description);
        this.updateMetaProperty('og:image', pageData.image || this.defaultMeta.image);
        this.updateMetaProperty('og:url', window.location.href);
        
        // Twitter Card
        this.updateMetaTag('twitter:title', pageData.title || this.defaultMeta.title, 'name');
        this.updateMetaTag('twitter:description', pageData.description || this.defaultMeta.description, 'name');
        this.updateMetaTag('twitter:image', pageData.image || this.defaultMeta.image, 'name');
        
        // Canonical
        this.updateCanonical();
    }
    
    updateMetaTag(name, content, attribute = 'name') {
        let meta = document.querySelector(`meta[${attribute}="${name}"]`);
        if (!meta) {
            meta = document.createElement('meta');
            meta.setAttribute(attribute, name);
            document.head.appendChild(meta);
        }
        meta.content = content;
    }
    
    updateMetaProperty(property, content) {
        this.updateMetaTag(property, content, 'property');
    }
    
    updateCanonical() {
        let canonical = document.querySelector('link[rel="canonical"]');
        if (!canonical) {
            canonical = document.createElement('link');
            canonical.rel = 'canonical';
            document.head.appendChild(canonical);
        }
        canonical.href = window.location.origin + window.location.pathname;
    }
}
```

### 6.2 Schema.org para Navegación

```javascript
// Agregar Schema.org dinámicamente
function addNavigationSchema() {
    const schema = {
        "@context": "https://schema.org",
        "@type": "SiteNavigationElement",
        "name": "Navegación Principal",
        "url": window.location.origin,
        "hasPart": [
            {
                "@type": "WebPage",
                "name": "Inicio",
                "url": window.location.origin + "/"
            },
            {
                "@type": "CollectionPage",
                "name": "Productos",
                "url": window.location.origin + "/productos"
            },
            {
                "@type": "ItemList",
                "name": "Experiencias",
                "url": window.location.origin + "/experiencias"
            },
            {
                "@type": "AboutPage",
                "name": "Nosotros",
                "url": window.location.origin + "/nosotros"
            },
            {
                "@type": "ContactPage",
                "name": "Contacto",
                "url": window.location.origin + "/contacto"
            }
        ]
    };
    
    const script = document.createElement('script');
    script.type = 'application/ld+json';
    script.textContent = JSON.stringify(schema);
    document.head.appendChild(script);
}
```

### 6.3 Breadcrumbs SEO

```javascript
// breadcrumbs.js
class BreadcrumbManager {
    constructor() {
        this.container = document.getElementById('breadcrumbs');
    }
    
    generate(path) {
        const parts = path.split('/').filter(Boolean);
        const breadcrumbs = [{
            name: 'Inicio',
            url: '/',
            position: 1
        }];
        
        let currentPath = '';
        parts.forEach((part, index) => {
            currentPath += '/' + part;
            breadcrumbs.push({
                name: this.formatName(part),
                url: currentPath,
                position: index + 2
            });
        });
        
        this.render(breadcrumbs);
        this.addSchema(breadcrumbs);
    }
    
    formatName(slug) {
        return slug
            .split('-')
            .map(word => word.charAt(0).toUpperCase() + word.slice(1))
            .join(' ');
    }
    
    render(breadcrumbs) {
        if (!this.container) return;
        
        const html = `
            <nav aria-label="Breadcrumb">
                <ol class="breadcrumb">
                    ${breadcrumbs.map((item, index) => `
                        <li class="breadcrumb-item">
                            ${index < breadcrumbs.length - 1 
                                ? `<a href="${item.url}">${item.name}</a>` 
                                : `<span aria-current="page">${item.name}</span>`
                            }
                        </li>
                    `).join('')}
                </ol>
            </nav>
        `;
        
        this.container.innerHTML = html;
    }
    
    addSchema(breadcrumbs) {
        const schema = {
            "@context": "https://schema.org",
            "@type": "BreadcrumbList",
            "itemListElement": breadcrumbs.map(item => ({
                "@type": "ListItem",
                "position": item.position,
                "name": item.name,
                "item": window.location.origin + item.url
            }))
        };
        
        const script = document.createElement('script');
        script.type = 'application/ld+json';
        script.textContent = JSON.stringify(schema);
        document.head.appendChild(script);
    }
}
```

---

## 7. Responsive Design

### 7.1 Mobile Navigation

```javascript
// mobile-nav.js
class MobileNavigation {
    constructor() {
        this.toggle = document.querySelector('.mobile-menu-toggle');
        this.nav = document.querySelector('.nav-links');
        this.isOpen = false;
        
        this.init();
    }
    
    init() {
        if (!this.toggle || !this.nav) return;
        
        this.toggle.addEventListener('click', () => this.toggleMenu());
        
        // Cerrar menú al hacer click fuera
        document.addEventListener('click', (e) => {
            if (this.isOpen && !e.target.closest('.main-nav')) {
                this.closeMenu();
            }
        });
        
        // Cerrar menú al cambiar tamaño de ventana
        window.addEventListener('resize', () => {
            if (window.innerWidth > 768 && this.isOpen) {
                this.closeMenu();
            }
        });
    }
    
    toggleMenu() {
        this.isOpen ? this.closeMenu() : this.openMenu();
    }
    
    openMenu() {
        this.isOpen = true;
        this.nav.style.display = 'flex';
        this.toggle.setAttribute('aria-expanded', 'true');
        this.toggle.classList.add('active');
        
        // Animación
        requestAnimationFrame(() => {
            this.nav.classList.add('open');
        });
    }
    
    closeMenu() {
        this.isOpen = false;
        this.toggle.setAttribute('aria-expanded', 'false');
        this.toggle.classList.remove('active');
        this.nav.classList.remove('open');
        
        // Esperar animación antes de ocultar
        setTimeout(() => {
            if (!this.isOpen) {
                this.nav.style.display = '';
            }
        }, 300);
    }
}
```

### 7.2 CSS Mobile First

```css
/* Mobile Navigation Styles */
@media (max-width: 767px) {
    .nav-links {
        position: fixed;
        top: 70px;
        left: 0;
        right: 0;
        background-color: #ffffff;
        flex-direction: column;
        padding: 1rem;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        transform: translateY(-100%);
        transition: transform 0.3s ease;
    }
    
    .nav-links.open {
        transform: translateY(0);
    }
    
    .mobile-menu-toggle {
        display: block;
        background: none;
        border: none;
        padding: 0.5rem;
        cursor: pointer;
    }
    
    .hamburger {
        display: block;
        width: 25px;
        height: 3px;
        background-color: var(--arcatierra-verde-tipografia);
        position: relative;
        transition: background-color 0.3s;
    }
    
    .hamburger::before,
    .hamburger::after {
        content: '';
        position: absolute;
        width: 100%;
        height: 100%;
        background-color: inherit;
        transition: transform 0.3s;
    }
    
    .hamburger::before {
        transform: translateY(-8px);
    }
    
    .hamburger::after {
        transform: translateY(8px);
    }
    
    .mobile-menu-toggle.active .hamburger {
        background-color: transparent;
    }
    
    .mobile-menu-toggle.active .hamburger::before {
        transform: rotate(45deg);
    }
    
    .mobile-menu-toggle.active .hamburger::after {
        transform: rotate(-45deg);
    }
}
```

---

## 8. Testing y Validación

### 8.1 Checklist de Validación SEO

```javascript
// seo-validator.js
class SEOValidator {
    constructor() {
        this.errors = [];
        this.warnings = [];
    }
    
    validate() {
        this.errors = [];
        this.warnings = [];
        
        // Validar título
        this.validateTitle();
        
        // Validar descripción
        this.validateDescription();
        
        // Validar encabezados
        this.validateHeadings();
        
        // Validar imágenes
        this.validateImages();
        
        // Validar enlaces
        this.validateLinks();
        
        // Validar Schema.org
        this.validateSchema();
        
        return {
            passed: this.errors.length === 0,
            errors: this.errors,
            warnings: this.warnings
        };
    }
    
    validateTitle() {
        const title = document.title;
        
        if (!title) {
            this.errors.push('Falta etiqueta <title>');
        } else if (title.length < 30) {
            this.warnings.push('Título muy corto (menos de 30 caracteres)');
        } else if (title.length > 60) {
            this.warnings.push('Título muy largo (más de 60 caracteres)');
        }
    }
    
    validateDescription() {
        const desc = document.querySelector('meta[name="description"]');
        
        if (!desc) {
            this.errors.push('Falta meta description');
        } else if (desc.content.length < 120) {
            this.warnings.push('Descripción muy corta (menos de 120 caracteres)');
        } else if (desc.content.length > 155) {
            this.warnings.push('Descripción muy larga (más de 155 caracteres)');
        }
    }
    
    validateHeadings() {
        const h1s = document.querySelectorAll('h1');
        
        if (h1s.length === 0) {
            this.errors.push('Falta etiqueta H1');
        } else if (h1s.length > 1) {
            this.warnings.push('Múltiples H1 encontrados');
        }
        
        // Validar jerarquía
        const headings = document.querySelectorAll('h1, h2, h3, h4, h5, h6');
        let lastLevel = 0;
        
        headings.forEach(heading => {
            const level = parseInt(heading.tagName[1]);
            if (level - lastLevel > 1) {
                this.warnings.push(`Salto en jerarquía de encabezados: H${lastLevel} a H${level}`);
            }
            lastLevel = level;
        });
    }
    
    validateImages() {
        const images = document.querySelectorAll('img');
        
        images.forEach((img, index) => {
            if (!img.alt) {
                this.errors.push(`Imagen #${index + 1} sin atributo alt`);
            }
            
            if (!img.width || !img.height) {
                this.warnings.push(`Imagen #${index + 1} sin dimensiones especificadas`);
            }
        });
    }
    
    validateLinks() {
        const links = document.querySelectorAll('a');
        
        links.forEach((link, index) => {
            if (!link.href) {
                this.errors.push(`Enlace #${index + 1} sin href`);
            }
            
            if (link.target === '_blank' && !link.rel?.includes('noopener')) {
                this.warnings.push(`Enlace externo #${index + 1} sin rel="noopener"`);
            }
        });
    }
    
    validateSchema() {
        const schemas = document.querySelectorAll('script[type="application/ld+json"]');
        
        if (schemas.length === 0) {
            this.warnings.push('No se encontró Schema.org');
        }
        
        schemas.forEach((schema, index) => {
            try {
                JSON.parse(schema.textContent);
            } catch (error) {
                this.errors.push(`Schema.org #${index + 1} tiene JSON inválido`);
            }
        });
    }
}
```

### 8.2 Performance Monitor

```javascript
// performance-monitor.js
class PerformanceMonitor {
    constructor() {
        this.metrics = {};
    }
    
    measure() {
        // Navigation Timing API
        const navigation = performance.getEntriesByType('navigation')[0];
        
        if (navigation) {
            this.metrics = {
                // Tiempo total de carga
                loadTime: navigation.loadEventEnd - navigation.fetchStart,
                
                // Tiempo hasta primer byte
                ttfb: navigation.responseStart - navigation.fetchStart,
                
                // Tiempo de descarga del documento
                downloadTime: navigation.responseEnd - navigation.responseStart,
                
                // Tiempo de procesamiento DOM
                domProcessing: navigation.domComplete - navigation.domLoading,
                
                // Tiempo hasta interacción
                tti: navigation.domInteractive - navigation.fetchStart
            };
        }
        
        // Core Web Vitals
        this.measureCoreWebVitals();
        
        return this.metrics;
    }
    
    measureCoreWebVitals() {
        // Largest Contentful Paint (LCP)
        new PerformanceObserver((entryList) => {
            const entries = entryList.getEntries();
            const lastEntry = entries[entries.length - 1];
            this.metrics.lcp = lastEntry.renderTime || lastEntry.loadTime;
        }).observe({ entryTypes: ['largest-contentful-paint'] });
        
        // First Input Delay (FID)
        new PerformanceObserver((entryList) => {
            const firstInput = entryList.getEntries()[0];
            this.metrics.fid = firstInput.processingStart - firstInput.startTime;
        }).observe({ entryTypes: ['first-input'] });
        
        // Cumulative Layout Shift (CLS)
        let clsValue = 0;
        new PerformanceObserver((entryList) => {
            for (const entry of entryList.getEntries()) {
                if (!entry.hadRecentInput) {
                    clsValue += entry.value;
                }
            }
            this.metrics.cls = clsValue;
        }).observe({ entryTypes: ['layout-shift'] });
    }
    
    report() {
        console.table(this.metrics);
        
        // Enviar a analytics si está configurado
        if (window.gtag) {
            Object.entries(this.metrics).forEach(([metric, value]) => {
                window.gtag('event', 'performance', {
                    metric_name: metric,
                    value: Math.round(value)
                });
            });
        }
    }
}
```

---

## 9. Checklist de Implementación

### 9.1 Pre-implementación
- [ ] Revisar paleta de colores oficial
- [ ] Preparar imágenes optimizadas (WebP con fallback)
- [ ] Configurar variables CSS
- [ ] Crear estructura de carpetas

### 9.2 Header
- [ ] Implementar estructura HTML semántica
- [ ] Agregar navegación con ARIA labels
- [ ] Implementar sistema de autenticación
- [ ] Agregar navegación móvil responsive
- [ ] Validar accesibilidad con lector de pantalla

### 9.3 Footer
- [ ] Implementar estructura con microdatos
- [ ] Agregar Schema.org Organization
- [ ] Incluir enlaces legales
- [ ] Optimizar para móviles
- [ ] Validar contraste de colores

### 9.4 SEO
- [ ] Configurar meta tags dinámicos
- [ ] Implementar Schema.org
- [ ] Agregar breadcrumbs
- [ ] Validar con herramientas SEO
- [ ] Configurar Google Search Console

### 9.5 Performance
- [ ] Minimizar CSS/JS
- [ ] Implementar lazy loading
- [ ] Configurar caché del navegador
- [ ] Optimizar imágenes
- [ ] Medir Core Web Vitals

### 9.6 Testing
- [ ] Probar en múltiples navegadores
- [ ] Validar en dispositivos móviles
- [ ] Ejecutar auditoría de Lighthouse
- [ ] Validar HTML con W3C
- [ ] Probar con usuarios reales

---

## 10. Troubleshooting

### Problema: Header no se mantiene sticky en iOS
```css
/* Solución */
.main-header {
    position: -webkit-sticky;
    position: sticky;
    top: 0;
    z-index: 1000;
    transform: translateZ(0); /* Force GPU acceleration */
}
```

### Problema: LocalStorage no disponible en modo privado
```javascript
// Solución: Storage wrapper con fallback
class SafeStorage {
    constructor() {
        this.isAvailable = this.checkAvailability();
        this.memory = {};
    }
    
    checkAvailability() {
        try {
            const test = '__storage_test__';
            localStorage.setItem(test, test);
            localStorage.removeItem(test);
            return true;
        } catch (e) {
            return false;
        }
    }
    
    getItem(key) {
        return this.isAvailable 
            ? localStorage.getItem(key) 
            : this.memory[key];
    }
    
    setItem(key, value) {
        if (this.isAvailable) {
            localStorage.setItem(key, value);
        } else {
            this.memory[key] = value;
        }
    }
}
```

### Problema: FOUC (Flash of Unstyled Content)
```html
<!-- Solución: Critical CSS inline -->
<style>
    /* Critical CSS */
    .main-header { 
        background: #fff; 
        height: 70px; 
    }
    body { 
        margin: 0; 
        font-family: system-ui; 
    }
</style>
<link rel="preload" href="/css/main.css" as="style">
<link rel="stylesheet" href="/css/main.css">
```

---

## Conclusión

Este manual proporciona una implementación completa y optimizada del sistema de header/footer unificado para ArcaTierra. Siguiendo estas guías, se garantiza:

1. **Consistencia Visual**: Misma experiencia en todas las páginas
2. **SEO Optimizado**: Mejor posicionamiento en buscadores
3. **Performance**: Carga rápida y eficiente
4. **Accesibilidad**: Navegación para todos los usuarios
5. **Mantenibilidad**: Código limpio y documentado

Para soporte adicional o consultas técnicas, contactar al equipo de desarrollo en dev@arcatierra.com.

---

**Documento creado por:** Equipo Técnico ArcaTierra  
**Fecha:** Junio 2025  
**Versión:** 1.0