# Google Identity Services Setup (GIS Moderno)

**Tu tarea:** Obtener `GOOGLE_CLIENT_ID` para Google One Tap

**Tiempo:** ~10 minutos

## Decisión práctica (tu arquitectura real)

**Objetivo definido:**
- Los usuarios entran desde Angular (`arcadia`) y ven Google login friendly (GIS).
- Keycloak sigue siendo el SSO central para Google/Facebook/OTP/email.

**Hoy crea 2 OAuth Client IDs (Web application):**

1. `arcadia-gis-frontend`  
   Para Google GIS en Angular (One Tap / botón Google).
2. `keycloak-google-broker`  
   Para federación Google dentro de Keycloak.

Esto es el punto justo: usable para startup, escalable, sin sobreingeniería.

---

## Comparativa Azure → Google Cloud

```
AZURE                          GOOGLE CLOUD
─────────────────────────────────────────────────────────
Organization (Entra ID)    →   Organization (dominio)
Tenant                     →   No existe equivalente directo
Subscription               →   Billing Account
Resource Group (RG)        →   Project  ← AQUÍ COMIENZAS
Entra ID (Azure AD)        →   Google Identity Platform
App Registration           →   OAuth 2.0 Client ID  ← LO QUE CREARÁS
Service Principal          →   Service Account
```

**En Azure harías:**
```
Entra ID → App Registrations → New Registration
```

**En Google Cloud haces:**
```
Project → APIs & Services → Credentials → Create OAuth 2.0 Client ID
```

Son lo mismo conceptualmente. El **Project** es suficiente (no necesitas Org ni Billing):

```
Tu cuenta Gmail personal
└── Project: "ADA Marketplace"   ← Equivale a RG + App Registration
    ├── APIs habilitadas
    ├── OAuth Credentials (Client IDs)  ← Lo que necesitas
    └── Service Accounts (no necesitas por ahora)
```

**¿Necesitas pagar?** No.

| Servicio | Costo |
|---|---|
| Google Identity Services (GIS) | Gratis, sin límite |
| OAuth 2.0 Credentials | Gratis |
| Consent Screen | Gratis |
| Cuenta Google Cloud | Gratis (pide tarjeta pero NO cobra para esto) |

---

---

## Step 1: Google Cloud Console

```
1. Abre: https://console.cloud.google.com/
2. Login con tu cuenta personal (gmail)
3. Te va a pedir seleccionar o crear proyecto
```

---

## Step 2: Crear Proyecto (si no existe)

```
En Google Cloud Console:
├─ Click en "Select a Project" (arriba a la izquierda)
├─ Click en "NEW PROJECT"
├─ Name: "ADA Marketplace"
├─ Organization: (dejar vacío)
├─ Location: (default)
└─ CREATE
```

**Espera a que termine (2-3 min)**

---

## Step 3: Habilitar Google Identity Services API

```
1. Click left sidebar -> "APIs & Services" 
2. Eneble OAuth consent screen (si no lo hiciste antes)
Choose "External" → Create → Completa campos básicos (App name, email) → Save and Continue → Skip scopes → Save and Continue → Add test user (tu-email@gmail.com) → Save and Continue → Back to Dashboard


(Espera a que se habilite... ~1 min)
```

---

## Step 4: Crear OAuth 2.0 Credentials (los 2 que necesitas ahora)

```
1. Click en "Credentials" (left sidebar)
2. Click en botón "+ CREATE CREDENTIALS" (arriba)
3. Select "OAuth 2.0 Client IDs"

   ⚠️  Te dirá "You need to configure the OAuth consent screen first"
   → Click en "CONFIGURE CONSENT SCREEN"
```

---

## Step 5: Configurar OAuth Consent Screen

```
┌─ User Type
│  ├─ Selecciona: "External" o publico en español
│  └─ Click CREATE
│
├─ OAuth consent screen form:
│  ├─ App name: "ADA Marketplace"
│  ├─ User support email: tu-email@gmail.com
│  ├─ Developer contact: tu-email@gmail.com
│  └─ Click "SAVE AND CONTINUE"
│
├─ Scopes (dejar default, SKIP)
│  └─ Click "SAVE AND CONTINUE"
│
├─ Test users (IMPORTANTE)
│  ├─ Click "ADD USERS"
│  ├─ Email: tu-email@gmail.com (el mismo con que logineas)
│  ├─ Click ADD
│  └─ Click "SAVE AND CONTINUE"
│
└─ Review y click "BACK TO DASHBOARD"
```

---

## Step 6: Crear Cliente #1 (Frontend GIS) — `arcadia-gis-frontend`

```
Back en "Credentials":

1. Click "+ CREATE CREDENTIALS" nuevamente
2. Select "OAuth 2.0 Client IDs"
3. Application type: "Web application"
4. Name: "arcadia-gis-frontend"

5. Authorized JavaScript origins:
  ├─ http://localhost:4200
  ├─ http://localhost:4202
  ├─ https://arcadia.sotobotero.com
  └─ https://ada.sotobotero.com   (solo si ahí renderizas login con Google)

6. Authorized redirect URIs:
  ├─ http://localhost:4200/auth/callback
  ├─ http://localhost:4202/auth/callback
  └─ https://arcadia.sotobotero.com/auth/callback

  Nota: para GIS One Tap puro, los origins son lo crítico.
  Los redirect URIs aplican si usas code flow por redirect.

7. Click "CREATE"
```

---

## Step 7: Crear Cliente #2 (Broker Keycloak) — `keycloak-google-broker`

```
1. Click "+ CREATE CREDENTIALS"
2. Select "OAuth 2.0 Client IDs"
3. Application type: "Web application"
4. Name: "keycloak-google-broker"

5. Authorized JavaScript origins:
  ├─ https://sso.sotobotero.com
  └─ http://localhost:8082   (solo si pruebas Keycloak local)

6. Authorized redirect URIs:
  ├─ https://sso.sotobotero.com/realms/arcadia/broker/google/endpoint
  └─ http://localhost:8082/realms/arcadia/broker/google/endpoint   (solo si pruebas Keycloak local)

7. Click "CREATE"

Regla crítica de Google:
- `Authorized JavaScript origins` = solo origen (`scheme://host[:port]`) sin path.
- `Authorized redirect URIs` = URI completa con path de callback.
```

---

## Step 8: Copiar credenciales

```
Verás una modal: "OAuth client created"

┌─────────────────────────────────────┐
│ Client ID                           │
│ xxxxxx.apps.googleusercontent.com   │ ← COPIAR ESTO
│                                     │
│ Client secret                       │
│ xxxxxx                              │ ← COPIAR ESTO TAMBIÉN
└─────────────────────────────────────┘

Regla rápida:
- `arcadia-gis-frontend`: usa **Client ID** en Angular.
- `keycloak-google-broker`: usa **Client ID + Client Secret** en Keycloak.
```

---

## Step 9: Guardar variables

```bash
# Angular (GIS)
ARCADIA_GIS_CLIENT_ID=xxxxx.apps.googleusercontent.com

# Keycloak broker
KEYCLOAK_GOOGLE_BROKER_CLIENT_ID=xxxxx.apps.googleusercontent.com
KEYCLOAK_GOOGLE_BROKER_CLIENT_SECRET=xxxxx
```

```bash
# En Keycloak (Identity Providers -> Google), cargar por entorno:
# - Client ID
# - Client Secret
# - Default Scopes: openid profile email
```

```text
Nota microfrontends (host:4200, booking:4202):
- Sí van en el cliente GIS de frontend (origins).
- No van en el callback del cliente broker (que apunta a /broker/google/endpoint).
- Si Google muestra: "Los URI no deben contener una ruta", moviste un callback al campo de origins.
```

---

## Step 10: Verificar credenciales en Google

```
En Google Cloud Console → Credentials:

Deberías ver exactamente 2 clientes:
┌──────────────────────────────────────────────────────────────┐
│ arcadia-gis-frontend   -> origins 4200/4202 + dominios      │
│ keycloak-google-broker -> redirect sso.../broker/google/... │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 11: Flujo recomendado hoy (simple y robusto)

```text
Usuario entra por Angular (host/booking)
Angular muestra GIS (One Tap / botón)
Autenticación final se centraliza en Keycloak (broker + demás IDPs)
Backend confía en tokens/claims alineados con Keycloak
```

---

## Test: Google One Tap en Local

```html
<!-- test.html -->
<!DOCTYPE html>
<html>
<head>
  <script src="https://accounts.google.com/gsi/client" async defer></script>
  <script>
    window.onload = function() {
      google.accounts.id.initialize({
        client_id: "YOUR_CLIENT_ID_HERE",
        callback: (response) => {
          console.log("JWT:", response.credential);
          alert("¡Login con Google funcionó!");
        }
      });
      
      google.accounts.id.prompt();
    };
  </script>
</head>
<body>
  <h1>Test Google One Tap</h1>
  <p>Si ves una ventana emergente arriba a la derecha = ¡OK!</p>
</body>
</html>
```

---

## Checklist

- [ ] Google Cloud project creado
- [ ] Google Identity Services API habilitado
- [ ] OAuth consent screen configurado
- [ ] OAuth client `arcadia-gis-frontend` creado
- [ ] OAuth client `keycloak-google-broker` creado
- [ ] Client IDs + Secret guardados
- [ ] GIS probado en Angular (4200/4202)
- [ ] Keycloak Google IDP configurado y probado

---

## Una vez tengas ambos Client IDs

Avísame y procedo con:
1. Configurar Google IDP en Keycloak (con broker client)
2. Ajustar variables en Angular para GIS client
3. Probar end-to-end login Google + emisión de token
