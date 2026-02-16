# Plan: Integrar Coordination Settings en Evolution Manager v2

**Fecha:** 2025-02-15  
**Rama API:** `coordinador-bot-chat` (Evolution API)  
**Proyecto:** `/home/msosa/evolution-manager-v2`  

---

## Contexto

Evolution API (rama `coordinador-bot-chat`) ahora soporta un campo `coordinationSettings` (JSON) en la configuración de Chatwoot por instancia. Este campo se envía y recibe via los endpoints existentes:

- **POST** `/chatwoot/set/{instanceName}` — crear/actualizar (ya incluye `coordinationSettings`)
- **GET** `/chatwoot/find/{instanceName}` — leer (ya devuelve `coordinationSettings`)

La UI (Evolution Manager v2) necesita exponer estos 4 toggles en la página de Chatwoot.

### Campos de coordinationSettings

| Campo | Tipo | Default API | Descripción |
|-------|------|-------------|-------------|
| `checkAgent` | `boolean` | `true` | Verificar si hay agente humano antes de activar bot |
| `autoPause` | `boolean` | `true` | Pausar bot cuando agente responde desde Chatwoot |
| `autoResolve` | `boolean` | `true` | Resolver conversación al terminar flujo del bot |
| `manageEnabled` | `boolean` | `true` | Habilitar endpoint /chatbot/manage |

---

## Stack del Manager v2

- **React 18** + **TypeScript** + **Vite**
- **UI**: shadcn/ui (Radix) + TailwindCSS
- **Forms**: react-hook-form + zod
- **State**: @tanstack/react-query
- **i18n**: i18next (4 idiomas: pt-BR, es-ES, en-US, fr-FR)
- **API**: axios via `src/lib/queries/`

---

## Archivos a modificar (6 archivos, ~50 líneas total)

### 1. Tipos — `src/types/evolution.types.ts`

**Cambio:** Agregar `coordinationSettings` al tipo `Chatwoot`.

```typescript
export type Chatwoot = {
  // ... campos existentes ...
  ignoreJids?: string[];
  coordinationSettings?: {
    checkAgent?: boolean;
    autoPause?: boolean;
    autoResolve?: boolean;
    manageEnabled?: boolean;
  };
};
```

**Impacto:** ~7 líneas. El tipo `FetchChatwoot` en `src/lib/queries/chatwoot/types.ts` es `any`, no requiere cambios.

---

### 2. Traducciones — 4 archivos JSON

**Cambio:** Agregar keys de i18n bajo `chatwoot.form.coordination*` en los 4 idiomas.

**Ejemplo es-ES:**
```json
{
  "chatwoot": {
    "form": {
      "coordinationTitle": "Coordinación Bot-Agente",
      "coordinationDescription": "Configuración de comportamiento entre bots y agentes humanos en Chatwoot",
      "checkAgent": {
        "label": "Verificar Agente",
        "description": "Verificar si hay un agente humano asignado antes de activar el bot"
      },
      "autoPause": {
        "label": "Auto-Pausar Bot",
        "description": "Pausar el bot automáticamente cuando un agente responde desde Chatwoot"
      },
      "autoResolve": {
        "label": "Auto-Resolver Conversación",
        "description": "Resolver la conversación en Chatwoot cuando el flujo del bot termina"
      },
      "manageEnabled": {
        "label": "Endpoint de Gestión",
        "description": "Habilitar el endpoint /chatbot/manage para control externo (Typebot HTTP Request, etc.)"
      }
    }
  }
}
```

**Impacto:** ~20 líneas por idioma × 4 idiomas.

---

### 3. Página Chatwoot — `src/pages/instance/Chatwoot/index.tsx`

**Cambios:**

#### a) Zod schema — agregar coordinationSettings

```typescript
const formSchema = z.object({
  // ... campos existentes ...
  coordinationSettings: z.object({
    checkAgent: z.boolean().optional(),
    autoPause: z.boolean().optional(),
    autoResolve: z.boolean().optional(),
    manageEnabled: z.boolean().optional(),
  }).optional(),
});
```

#### b) Default values — agregar defaults

```typescript
defaultValues: {
  // ... campos existentes ...
  coordinationSettings: {
    checkAgent: true,
    autoPause: true,
    autoResolve: true,
    manageEnabled: true,
  },
},
```

#### c) useEffect (load) — mapear coordinationSettings desde API

```typescript
coordinationSettings: {
  checkAgent: chatwoot.coordinationSettings?.checkAgent ?? true,
  autoPause: chatwoot.coordinationSettings?.autoPause ?? true,
  autoResolve: chatwoot.coordinationSettings?.autoResolve ?? true,
  manageEnabled: chatwoot.coordinationSettings?.manageEnabled ?? true,
},
```

#### d) onSubmit — incluir en el payload

```typescript
coordinationSettings: data.coordinationSettings,
```

#### e) JSX — sección visual con 4 switches

Agregar después de `autoCreate` y antes del botón de guardar:

```tsx
<Separator className="my-4" />
<h4 className="mb-1 text-md font-medium">{t("chatwoot.form.coordinationTitle")}</h4>
<p className="text-sm text-muted-foreground mb-2">{t("chatwoot.form.coordinationDescription")}</p>
<div className="mx-4 space-y-2 divide-y [&>*]:px-4 [&>*]:py-2">
  <FormSwitch
    name="coordinationSettings.checkAgent"
    label={t("chatwoot.form.checkAgent.label")}
    className="w-full justify-between"
    helper={t("chatwoot.form.checkAgent.description")}
  />
  <FormSwitch
    name="coordinationSettings.autoPause"
    label={t("chatwoot.form.autoPause.label")}
    className="w-full justify-between"
    helper={t("chatwoot.form.autoPause.description")}
  />
  <FormSwitch
    name="coordinationSettings.autoResolve"
    label={t("chatwoot.form.autoResolve.label")}
    className="w-full justify-between"
    helper={t("chatwoot.form.autoResolve.description")}
  />
  <FormSwitch
    name="coordinationSettings.manageEnabled"
    label={t("chatwoot.form.manageEnabled.label")}
    className="w-full justify-between"
    helper={t("chatwoot.form.manageEnabled.description")}
  />
</div>
```

**Impacto:** ~40 líneas en el componente.

---

## Lo que NO necesita cambios

| Archivo | Razón |
|---------|-------|
| `src/lib/queries/chatwoot/fetchChatwoot.ts` | Ya devuelve `response.data` completo |
| `src/lib/queries/chatwoot/manageChatwoot.tsx` | Ya envía `data: Chatwoot` completo al POST |
| `src/lib/queries/chatwoot/types.ts` | Es `any`, no restringe |
| `src/routes/index.tsx` | No hay nueva ruta |
| `src/components/sidebar.tsx` | No hay nuevo menú |
| `src/lib/queries/api.ts` | No cambia la API base |

---

## Orden de implementación

1. **Tipo** `Chatwoot` en `evolution.types.ts` (~2 min)
2. **Traducciones** en 4 archivos JSON (~5 min)
3. **Página** `Chatwoot/index.tsx` — schema, defaults, load, submit, JSX (~10 min)
4. **Verificar** con `npm run dev` y probar visualmente (~5 min)
5. **Commit** en rama nueva del manager

---

## Resultado esperado

La página de Chatwoot en el Manager mostrará una nueva sección **"Coordinación Bot-Agente"** con 4 toggles, debajo de las opciones existentes. Los valores se persisten via el endpoint `/chatwoot/set` existente y se leen via `/chatwoot/find`. Si el usuario no toca los toggles, se envían como `true` (defaults = coordinación completa).

---

## Consideraciones

- **Backward compatible**: Si la API no devuelve `coordinationSettings` (versión anterior), los defaults son `true` (gracias al `?? true` en el useEffect)
- **No requiere nueva ruta**: Todo se integra en la página Chatwoot existente
- **No requiere nueva API**: Usa los endpoints existentes de `/chatwoot/set` y `/chatwoot/find`
- **Validación zod**: El objeto es opcional, sus campos internos también
