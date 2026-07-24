# Mis Propinas

PWA (Progressive Web App) para que un equipo registre y controle propinas, membresías (PRO/PREMIUM) y venta de cursos, con paneles de cobro por período y un panel de administrador. Desplegada en Vercel (`proopinas.vercel.app`), backend en Supabase (Postgres + Auth).

## Estructura

Proyecto sin build ni bundler — todo se sirve tal cual:

- `index.html` — archivo único con HTML, `<style>` y `<script>` inline (auth, UI, lógica de negocio, exportación a Excel/PDF). Es intencionalmente monolítico; no se ha dividido en módulos.
- `sw.js` — service worker mínimo (cachea el shell de la app, versión de caché en `CACHE = 'mispropinas-vN'`; súbela al tocar el shell para invalidar cachés viejas).
- `manifest.json` — metadata de PWA (nombre, iconos, `start_url`/`scope` apuntando al dominio de producción).
- `icon-192.png`, `icon-512.png` — iconos de la PWA.

No hay `package.json`, linter, ni tests automatizados. Chart.js y el cliente de Supabase se cargan por CDN (`<script src="...">`), sin npm.

## Backend (Supabase)

Todo el acceso a datos es client-side con `sb.from('tabla').select/insert/update/delete()`. Tablas principales: `propinas`, `membresias`, `cursos`, `cobros`, `perfiles`, `configuracion`, `admins`.

- La clave `SUPABASE_KEY` en `index.html` es la **publishable key** (pensada para exponerse en el cliente); la seguridad real depende de las políticas RLS en Supabase, no del secreto de esa clave.
- Errores de operaciones no críticas (cargar ciclo, meta, permisos) se tragan silenciosamente con `catch(e){}` — no rompen la UI si falla una carga secundaria.
- Errores de operaciones que el usuario inició a propósito (guardar, eliminar, actualizar) sí deben mostrarse con `showToast(msg, true)`.

## Convención de datos: monto con signo

Las propinas no tienen una columna booleana de "es descuento": un monto negativo en `propinas.amount` **es** el descuento (ver `addTip()` / `renderTips()`). Cualquier validación o edición sobre montos de propinas debe permitir negativos (rechazar solo `0`/`NaN`), no asumir `amount > 0`.

## Estilo de código (JS)

- Vanilla JS, `async/await`, funciones globales declaradas con `function nombre(){}` (sin módulos, sin clases).
- camelCase para funciones y variables; kebab-case para ids del DOM (`f-name`, `e-amount`, `cobro-monto-prop`), que suelen coincidir con el propósito del elemento.
- Funciones cortas de una línea son comunes y aceptadas en este archivo (no se busca reformatear a multilínea por estilo).
- Todo el texto visible al usuario (labels, mensajes de error, toasts, confirmaciones) va en español, tono directo e informal.

## Estilo (CSS)

- Variables CSS en `:root` (`--bg`, `--text`, `--green`, `--radius`, etc.) con override en `@media(prefers-color-scheme:dark)` — no usar colores hardcodeados fuera de las variables si hay equivalente.
- Ajustes responsivos para teléfono están agrupados al final del `<style>` bajo `@media(max-width:600px)`, con el comentario explícito de que no deben afectar escritorio. Mantener esa separación al agregar estilos nuevos.

## Flujo de contribución (Git/PRs)

El historial antiguo de este repo son commits directos a `main` con mensajes genéricos (`Update index.html`). A partir de esta sesión se usa un flujo más formal para cambios hechos por Claude:

- Una rama por cambio, prefijo según el tipo: `fix/<descripción-corta>`, `docs/<descripción-corta>`.
- Commit con línea de resumen imperativa + cuerpo explicando el *porqué* (causa raíz del bug, no solo el qué).
- PR vía `gh pr create`, cuerpo en español con secciones `## Resumen` y `## Plan de prueba`.
- Cambios pequeños y acotados: un bug o mejora por PR, evitar mezclar refactors con fixes.

## Verificación

No hay suite de tests ni CI. La app requiere sesión autenticada de Supabase para probar flujos reales (login, guardar propina, etc.), así que la verificación típica es:

1. Abrir `index.html` en el navegador y revisar la consola por errores de JS/sintaxis.
2. Para lógica pura (validaciones, cálculos de fecha/periodo), probar la condición aislada en la consola del navegador cuando no se pueda ejercitar el flujo completo sin credenciales.
3. Dejar explícito en el PR qué se pudo probar y qué requiere prueba manual del usuario (todo lo que dependa de sesión autenticada, biometría, o el backend real de Supabase).
