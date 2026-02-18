[GUIA_INSTALACION_M365.md](https://github.com/user-attachments/files/25385515/GUIA_INSTALACION_M365.md)
# 🏢 GUÍA DE INSTALACIÓN — CITRO en Microsoft 365
**Tiempo total:** ~60 minutos | **Sin permisos de administrador de O365 necesarios***

> *Solo necesitas ser **Site Owner** en un sitio SharePoint y tener acceso a Azure AD para registrar una app.

---

## 📋 REQUISITOS PREVIOS

| Recurso | Cómo obtenerlo |
|---------|---------------|
| Cuenta `@uv.mx` (M365) | Ya la tienes |
| Sitio SharePoint (Site Owner) | Crear en sharepoint.com o pedirlo a TI |
| Azure App Registration | Pedirlo a TI **o** hacerlo tú (ver Paso 3) |

---

# PASO 1 — Crear Sitio SharePoint (5 min)

1. Ir a `https://uv.sharepoint.com` (o el tenant de la UV)
2. Click **+ Crear sitio** → **Sitio de equipo**
3. Configurar:
   ```
   Nombre:     CITRO Formularios
   URL:        CITRO-Formularios
   Privacidad: Privado
   ```
4. **Guardar la URL:**
   ```
   https://TU_TENANT.sharepoint.com/sites/CITRO-Formularios
   ```

---

# PASO 2 — Configurar SharePoint (15 min)

### Opción A — Script automático (recomendado)

```powershell
# Instalar módulo (solo 1 vez)
Install-Module PnP.PowerShell -Force

# Ejecutar script
.\sharepoint\Setup-SharePoint.ps1 `
  -SiteUrl "https://TU_TENANT.sharepoint.com/sites/CITRO-Formularios" `
  -AdminEmail "tu@uv.mx"
```

El script crea automáticamente la lista con 13 columnas y la biblioteca de PDFs con 5 carpetas.

---

### Opción B — Manual

**Crear Lista `SolicitudesCITRO`:**
Nuevo → Lista → Nombre: `SolicitudesCITRO`

Agregar columnas (Configuración de lista → Crear columna):

| Nombre Interno    | Tipo | Valores |
|-------------------|------|---------|
| Folio             | Línea de texto | — |
| TipoTramite       | Elección | Apoyo Académico / Aval Institucional / Apoyo a Terceros / Comité Tutorial / Solicitud Libre |
| NombreSolicitante | Línea de texto | — |
| EmailSolicitante  | Línea de texto | — |
| EmailUsuarioM365  | Línea de texto | — |
| Matricula         | Línea de texto | — |
| MontoSolicitado   | Moneda | — |
| MontoAutorizado   | Moneda | — |
| Estado            | Elección | Pendiente / En Revisión / Aprobado / Rechazado |
| DatosCompletos    | Varias líneas | — |
| NotasCT           | Varias líneas | — |
| URLPdf            | Hipervínculo | — |
| FechaSolicitud    | Fecha y hora | — |

**Crear Biblioteca `PDFs_Solicitudes`:**
Nuevo → Biblioteca de documentos → Crear 5 carpetas:
`01_Apoyo_Academico` / `02_Aval_Institucional` / `03_Apoyo_Terceros` / `04_Comite_Tutorial` / `05_Solicitud_Libre`

---

# PASO 3 — Registrar App en Azure AD (10 min)

1. Ir a **https://portal.azure.com**
2. Buscar → **App registrations** → **New registration**
3. Configurar:
   ```
   Name:  CITRO Gestión Académica
   
   Supported account types:
   ● Accounts in this organizational directory only (Single tenant)
   
   Redirect URI:
   Type: Single-page application (SPA)
   URI:  [URL donde vas a hospedar el frontend — ver Paso 5]
   ```
4. Click **Register**

5. En la página de la app, copiar:
   ```
   Application (client) ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ← CLIENT ID
   Directory (tenant) ID:   yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy  ← TENANT ID
   ```

6. **API permissions** → **Add a permission** → **Microsoft Graph** → **Delegated**:
   - `User.Read`
   - `Sites.ReadWrite.All`
   - `Calendars.ReadWrite`
   - `Mail.Send`

7. Click **Grant admin consent for [Organización]** ✅

---

# PASO 4 — Editar Configuración (5 min)

Abrir `frontend/config-m365.js` y editar:

```javascript
const CONFIG = {
    azure: {
        clientId: 'PEGAR_CLIENT_ID_AQUI',   // ← Del Paso 3
        tenantId: 'PEGAR_TENANT_ID_AQUI',   // ← Del Paso 3
    },
    sharepoint: {
        siteUrl:  'https://TU_TENANT.sharepoint.com/sites/CITRO-Formularios',
        tenant:   'TU_TENANT',
    },
    admins: [
        'director.citro@uv.mx',    // ← Emails con acceso al panel admin
        'secretario.ct@uv.mx',
    ],
    email: {
        adminEmail: 'consejo.tecnico@uv.mx',  // ← Recibe notificaciones
    }
};
```

---

# PASO 5 — Hospedar el Frontend (10 min)

### Opción A: GitHub Pages (gratis, recomendado)

```
1. github.com → New repository → citro-sistema → Public
2. Subir TODOS los archivos de la carpeta frontend/
3. Settings → Pages → Branch: main → Save
4. URL: https://TU-USUARIO.github.io/citro-sistema/
```

**Importante:** Volver a Azure AD y agregar esa URL como Redirect URI:
```
App registrations → Tu app → Authentication
→ Add URI → https://TU-USUARIO.github.io/citro-sistema/
→ Save
```

---

### Opción B: SharePoint como página

Si quieres que viva dentro del propio SharePoint:

1. Tu sitio SharePoint → **Páginas** → **Nueva página**
2. Agregar el Web Part **"Script Editor"** (o contactar TI para SPFx)
3. Pegar el contenido del `index.html`

---

### Opción C: Servidor institucional (FTP/cPanel)

Subir los archivos de `frontend/` vía FTP a tu servidor y agregar esa URL a Azure AD.

---

# PASO 6 — Configurar Power Automate (opcional, 15 min)

Power Automate envía notificaciones automáticas cuando cambia el estado de una solicitud.

### Flujo 1: Nueva solicitud → Email al CT

1. **make.powerautomate.com** → Nuevo flujo → "Cuando se crea un elemento" en SharePoint
2. Lista: `SolicitudesCITRO`
3. Acción: Enviar email (Outlook) al CT con los datos del item

### Flujo 2: Cambio de estado → Email al solicitante

1. Trigger: "Cuando se modifica un elemento" en `SolicitudesCITRO`
2. Condición: Campo `Estado` cambió
3. Acción: Enviar email a `EmailSolicitante` con el nuevo estado y monto autorizado

---

# PASO 7 — Verificar Instalación (5 min)

```
□ Abrir la URL del frontend
□ Click "Iniciar sesión con Microsoft"
□ Login con cuenta @uv.mx → Ver nombre e iniciales en header
□ Seleccionar "Apoyo Académico" → Llenar formulario → Enviar
□ Verificar PDF en SharePoint > PDFs_Solicitudes
□ Verificar item en lista SolicitudesCITRO
□ Recibir email de confirmación en Outlook
□ Click "Agregar a Outlook" → Evento creado en calendario
□ Si eres admin: Panel de Administración → Editar solicitud → Guardar
```

---

# 🔧 TROUBLESHOOTING

| Error | Causa | Solución |
|-------|-------|----------|
| `AADSTS50011` | Redirect URI no coincide | Agregar URL exacta en Azure AD → Authentication |
| `Access denied` | Admin consent no aprobado | `portal.azure.com` → API permissions → Grant consent |
| `Failed to fetch` | siteUrl incorrecto | Verificar URL en `config-m365.js` sin `/` al final |
| Login no aparece | clientId/tenantId incorrecto | Verificar `config-m365.js` líneas 4-5 |
| PDF no sube | `Sites.ReadWrite.All` falta | Verificar permisos en Azure AD |

---

# 📦 ARCHIVOS DEL PAQUETE

```
frontend/
├── index.html            → App principal (subir al hosting)
├── styles.css            → Diseño Fluent/M365
├── config-m365.js        ⚙️ EDITAR primero
├── auth-msal.js          → Login Azure AD
├── sharepoint.js         → Graph API + SharePoint
├── app-m365.js           → Lógica y PDF
├── admin-m365.js         → Panel administración
├── calendar-outlook.js   → Outlook Calendar
└── forms-data.js         → 5 formularios

sharepoint/
└── Setup-SharePoint.ps1  → Configuración automática

powerautomate/
└── CITRO_Flow.json       → Definición del flujo
```

---

# ✅ CHECKLIST COMPLETO

```
SHAREPOINT:
□ Sitio creado
□ Lista SolicitudesCITRO con columnas
□ Biblioteca PDFs_Solicitudes con carpetas

AZURE AD:
□ App Registration creada
□ clientId y tenantId copiados
□ 4 permisos agregados (User.Read, Sites.ReadWrite.All, Calendars.ReadWrite, Mail.Send)
□ Admin consent aprobado
□ Redirect URI configurada

CONFIGURACIÓN:
□ config-m365.js editado (clientId, tenantId, siteUrl, admins, adminEmail)

HOSTING:
□ Archivos subidos
□ URL en Azure AD coincide exactamente

PRUEBAS:
□ Login funciona
□ Formulario se envía
□ PDF en SharePoint
□ Item en lista
□ Email recibido en Outlook
□ Calendario Outlook funciona
□ Panel admin visible
□ Editar solicitud + monto autorizado funciona
```

---
**CITRO · Universidad Veracruzana · Sistema M365 v1.0**
