# Documentación completa para réplica de www.arcatierra.com

## 1. Estructura completa del sitio

### Arquitectura y páginas principales

El sitio de Arca Tierra está organizado alrededor de tres pilares principales que reflejan su modelo de negocio:

**Páginas principales:**
- **Homepage** - Página de inicio con misión y servicios principales
- **Nosotros** - Historia, equipo y filosofía de la empresa
- **Canastas Agroecológicas** - Sistema de suscripción y compra directa
- **Experiencias** - Turismo rural y gastronómico (públicas y privadas)
- **Tienda/Súper de Alimentos** - E-commerce con 13 categorías
- **Recetas** - Blog con recetas de cocina agroecológica
- **Restaurante Baldío** - Información del nuevo restaurante en Condesa
- **Contacto** - Información de contacto y ubicaciones
- **Prensa** - Sección para medios
- **Aliados** - Red de colaboradores

### Sistema de navegación

**Menú principal (Header):**
1. Nosotros
2. Restaurante Baldío
3. Prensa
4. Tienda/Súper de Alimentos (con dropdown de categorías)
5. Canastas Agroecológicas
6. Experiencias (con opciones públicas/privadas)
7. Recetas
8. ¿Cómo comprar?
9. Contacto

**Elementos adicionales del header:**
- Logo de Arca Tierra (superior izquierda)
- Verificador de código postal (para entregas)
- Carrito de compras (contador de items)
- Navegación responsive con menú hamburguesa para móvil

### Footer completo

**Estructura del footer:**
- **Sección de contacto por servicio:**
  - Canastas y Tienda: [email protected], +52 55 1051 5525
  - Experiencias: [email protected], +52 55 1051 5525, +52 1 56 1057 3296
  - Enlaces a todas las secciones principales
  
- **Información operativa:**
  - Bodega: Colonia San Miguel de Chapultepec I
  - Experiencias: Embarcadero Cuemanco, Xochimilco
  - Horarios: Lunes a Domingo, 9:00 - 18:00

- **Redes sociales:**
  - Instagram: @arcatierramx, @arca.tierra
  - Facebook: Arca Tierra (4,448 seguidores)
  - LinkedIn: arca tierra (337 seguidores)

### Elementos recurrentes

- **Call-to-Actions globales:**
  - "Ingresa tu código postal para verificar que llegamos a tu domicilio"
  - "Conocer más" (en secciones principales)
  - "Añadir al carrito" (productos)
  - "Reservar Ahora" (experiencias)
  - "Ver Calendario" (disponibilidad)

## 2. Diseño visual detallado

### Paleta de colores

**Colores principales identificados:**
- **Verde Natural** - Color principal para elementos de navegación y botones (coherente con identidad ecológica)
- **Blanco** - Fondo principal, crea espacios limpios y respirables
- **Negro/Gris Oscuro** - Textos principales y elementos de contraste
- **Tonos Tierra** - Marrones y ocres que complementan las imágenes agrícolas

*Nota: Los valores hexadecimales exactos requieren inspección directa del CSS*

### Sistema tipográfico

- **Familia tipográfica:** Sans-serif moderna y limpia
- **Jerarquía visual clara:**
  - Títulos principales (h1): Mayor tamaño, peso bold
  - Subtítulos (h2-h3): Tamaño medio, peso regular
  - Texto de párrafo: Tamaño estándar, alta legibilidad
  - Navegación: Peso regular, mayúsculas/minúsculas

### Espaciados y layout

- **Sistema responsive** con adaptación a múltiples dispositivos
- **Espaciado generoso** que crea sensación de limpieza
- **Grid system** estructurado en bloques bien definidos
- **Contenedores centrados** para organizar el contenido
- **Márgenes y paddings** consistentes en todo el sitio

### Efectos y animaciones

- **Transiciones suaves** en elementos de navegación
- **Estados hover** en botones y enlaces
- **Scroll behavior** suave entre secciones
- **Animaciones sutiles** que no distraen del contenido

### Componentes UI principales

- **Botones:** Esquinas redondeadas, colores coherentes con la marca
- **Cards/Tarjetas:** Para productos y experiencias
- **Formularios:** Diseño limpio con validaciones claras
- **Sistema de navegación:** Menú horizontal limpio y accesible

## 3. Contenido de cada sección

### Homepage

**Hero principal:**
- Título: "Somos una red local que promueve la buena alimentación a través de agricultura regenerativa y comercio justo"
- Subtítulo: "Desde 2009, nuestro equipo promueve la sostenibilidad y la agricultura campesina en términos productivos, sociales, ecológicos y económicos"

**Secciones principales:**

1. **Canastas Agroecológicas**
   - "Vendemos alimentos artesanales y canastas con la cosecha de la semana ya sea a través de suscripciones o con compra directa. Elige la que más te convenga y lleva a tu hogar frescura y salud."
   - CTA: "Conocer más"

2. **Experiencias Gastronómicas**
   - "Te ofrecemos experiencias de turismo rural y gastronómico en Xochimilco. Al ser parte de estas visitas ayudas a la preservación de saberes tradicionales agrícolas que hacen posible su existencia."
   - CTA: "Conocer más"

3. **Tienda en Línea**
   - "Nuestra tienda en línea tiene una deliciosa curaduría de alimentos orgánicos, regenerativos y 100% mexicanos ¡Complementa tu canasta de hortalizas y vegetales!"

### Productos/Tienda

**Categorías completas:**
- Canastas Agroecológicas
- Cacao, Café y Chocolate
- De la colmena
- Frutas y Verduras
- Despensa
- Huevo y Lácteos
- Maíz
- Mermeladas y Untables
- Para Regalar
- Pan, Pastas y Cereales
- Proteína Animal
- Otros

**Productos destacados:**
- **Canasta Individual:** 3.5 kg, $290.00 MXN
- **Canasta Básica Familiar:** 13 kg, $510.00 MXN
- Sistema de suscripción vs compra directa

### Experiencias

**Tipos de experiencias:**

1. **Amanecer en las Chinampas**
   - Duración: 3 horas
   - Precio: ~$65 USD por persona / $1,000 MXN
   - Incluye: Desayuno de 3 tiempos, recorrido en trajinera, tour guiado
   - Horarios: Principalmente domingos, 6:00-6:30 AM

2. **Brunch Chinampero**
   - Experiencia gastronómica con ingredientes locales
   - Tour de agricultura regenerativa incluido

3. **Experiencias Privadas**
   - Mínimo 10 personas
   - Desde $650 USD para el grupo
   - 9 experiencias distintas disponibles

**Políticas importantes:**
- Todas las ventas son finales
- No se aceptan mascotas
- Menores de 3 años no pagan
- Opciones vegetarianas siempre incluidas

### Nosotros

**Historia y fundación:**
- Fundador: Lucio Usobiaga (filósofo y emprendedor)
- Inicio: 2009
- Equipo: 32 personas
- Red: 40+ familias agricultoras de CDMX, Hidalgo, Puebla y Estado de México

**Misión:**
"Construir el futuro de la alimentación a través de agricultura regenerativa, preservando las chinampas y promoviendo el comercio justo"

## 4. Elementos interactivos

### Formularios

**Tipos implementados:**
- Formulario de contacto general
- Newsletter/suscripción
- Sistema de reservas para experiencias
- Verificador de código postal

**Campos típicos:**
- Nombre completo (requerido)
- Email (requerido, con validación)
- Teléfono (opcional)
- Mensaje/Comentarios
- Confirmación automática por email

### Sistema e-commerce

**Carrito de compras:**
- Botón "Añadir al carrito" en cada producto
- Gestión de cantidades
- Persistencia entre sesiones
- Integración con verificador de código postal

**Proceso de checkout:**
- Verificación de área de entrega
- Selección suscripción vs compra directa
- Opciones de entrega o recogida en bodega
- Métodos de pago: Todas las tarjetas, transferencias

**Gestión de inventario:**
- Estados: "AGOTADO", "Regresará pronto"
- Control automático de stock
- Rotación por temporadas agrícolas

### Sistema de reservas

**Funcionalidades:**
- Calendario de disponibilidad
- Selector de experiencias públicas/privadas
- Confirmación automática con detalles
- Recordatorios pre-visita
- Integración con pagos online

### Interacciones UI

**Componentes interactivos:**
- Navegación con dropdowns
- Filtros por categorías de productos
- Sistema de tabs/pestañas
- Acordeones para información detallada
- Modales para confirmaciones
- Sliders/carruseles de imágenes

## 5. Imágenes y recursos

### Imágenes principales

**Hero/Banner:**
- Fotografías panorámicas de chinampas al amanecer
- Vistas de canales de Xochimilco con volcanes
- Imágenes de alta calidad que transmiten la esencia del proyecto

**Productos:**
- Fotografías de canastas con vegetales frescos
- Imágenes individuales de productos artesanales
- Fotos de cultivos en las chinampas

**Experiencias:**
- Trajineras navegando por canales
- Chefs preparando alimentos
- Grupos disfrutando experiencias gastronómicas
- Agricultores trabajando en chinampas

### Recursos gráficos

**Iconografía:**
- Logo principal "arca tierra" (minúsculas)
- Iconos de servicios y características
- Iconos de redes sociales
- Símbolos de navegación

**Videos:**
- Tours virtuales de chinampas
- Contenido gastronómico
- Videos educativos sobre agricultura regenerativa
- Documentación de eventos especiales

### Optimización técnica

**Aspectos identificados:**
- Sitio desarrollado con React (create-react-app)
- Diseño completamente responsive
- Imágenes optimizadas para web
- Integración con sistemas de terceros

## Recomendaciones para implementación en Next.js 14

Para replicar este sitio con el stack moderno solicitado, recomiendo:

1. **Estructura de páginas con App Router** de Next.js 14
2. **TypeScript** para todo el código con tipado estricto
3. **Tailwind CSS** para replicar el sistema de diseño
4. **Componentes reutilizables** para cards, botones, formularios
5. **API Routes** para manejar formularios y pagos
6. **Optimización de imágenes** con next/image
7. **Internacionalización** preparada (español/inglés)
8. **Sistema de gestión de estado** para carrito y reservas
9. **Integración con pasarelas de pago** mexicanas
10. **SEO optimizado** con metadatos dinámicos

Esta documentación proporciona toda la información necesaria para crear una réplica exacta del sitio manteniendo su esencia, funcionalidad y valores de marca, pero implementada con tecnologías modernas.