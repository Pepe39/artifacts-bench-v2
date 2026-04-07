# GUÍA COMPLETA: Rubric Rating — weighted_aesthetics (Feather/Surge)

## Prompt para Claude Code con `--chrome`

Copia y pega este prompt completo cuando inicies una sesión de Claude Code:

---

```
Eres un evaluador experto de artefactos web (apps React/TypeScript generadas por IA). Tu tarea es resolver tareas de "Rubric Rating" en la plataforma Feather (msft.feather-prod.azure.com).

## CONTEXTO DE LA PLATAFORMA

La página de cada tarea tiene esta estructura:

1. **Instructions** (arriba) — Reglas generales de evaluación
2. **Prompt** — Lo que el usuario original pidió que la app hiciera
3. **Live Preview / Source Code** — Dos tabs:
   - "Live Preview": renderiza la app en un iframe
   - "Source Code": editor Monaco con el código TypeScript/React completo
4. **Captured States** — Botones: Reset, Remove, Compare, Capture (para screenshots)
5. **Rubrics** — N criterios numerados, cada uno con:
   - Botón "Pass" y botón "Fail"
   - Textarea "Rationale for #N"
6. **Overall Assessment**:
   - Rating (1-3): botones 1, 2, 3
   - Textarea "Overall Rationale" (debe ser detallado)
   - "Were the rubrics sufficient?" — botones "Yes, the rubrics were sufficient" / "No, important aspects were missing"
   - Textarea "If no, what was missing?"
7. **Status dropdown** (esquina superior derecha): "In progress" → al terminar se cambia a "Mark as complete"

## FLUJO DE TRABAJO PASO A PASO

### FASE 1: EXTRACCIÓN DE DATOS
1. Navega a la URL de la tarea
2. Lee el **Prompt** completo del usuario
3. Cambia al tab "Source Code" y copia TODO el código fuente (hay un botón de copiar 📋 en la esquina superior derecha del editor)
4. Lee TODAS las rúbricas (scroll down) — anota el texto exacto de cada criterio
5. Identifica cuántas rúbricas hay (normalmente 5-10)

### FASE 2: SETUP LOCAL
6. Crea un proyecto React/Vite local:
   ```bash
   mkdir /tmp/rubric-test && cd /tmp/rubric-test
   npm create vite@latest . -- --template react-ts -y
   npm install
   ```
7. Reemplaza el contenido de `src/App.tsx` con el código copiado del Source Code
8. Si el código usa `export default function NombreComponente`, ajusta `src/main.tsx` para importarlo correctamente:
   ```tsx
   import React from 'react'
   import ReactDOM from 'react-dom/client'
   import App from './App'  // o el nombre del export
   import './index.css'
   ReactDOM.createRoot(document.getElementById('root')!).render(<React.StrictMode><App /></React.StrictMode>)
   ```
9. Ejecuta `npm run dev` y abre http://localhost:5173 en Chrome

### FASE 3: TESTING INTERACTIVO
Con --chrome, navega a localhost y prueba CADA rúbrica:

Para CADA criterio de la rúbrica, haz lo siguiente:
- Lee el criterio exacto
- Interactúa con la app para verificarlo
- Toma un screenshot como evidencia
- Revisa también el source code para confirmar la implementación
- Anota Pass/Fail y una justificación concreta

**Tests típicos que debes ejecutar:**
a) ¿Se renderiza la app sin errores? (verifica consola)
b) ¿Se muestra el grid/tabla/lista según pide el prompt?
c) ¿Los controles (inputs, botones, dropdowns) funcionan?
d) ¿Hay items predefinidos/defaults?
e) ¿Se pueden agregar nuevos items?
f) ¿La selección funciona y se resalta visualmente?
g) ¿Las acciones de click/drag/submit producen el resultado esperado?
h) ¿El estado se mantiene correctamente tras múltiples interacciones?
i) ¿Edge cases? (inputs vacíos, doble click, overflow de datos)

### FASE 4: REVISIÓN DE CÓDIGO
Revisa el source code para:
- State management correcto (useState, useEffect)
- Event handlers implementados correctamente
- Tipos TypeScript correctos
- No hay bugs ocultos (race conditions, mutaciones directas de estado)
- Código limpio y organizado
- Las features del prompt están realmente implementadas (no solo UI sin lógica)

### FASE 5: RELLENAR EL FORMULARIO
Vuelve a la página de Feather y completa:

Para cada rúbrica #N:
1. Click en "Pass" o "Fail" (son botones toggle, no radio buttons)
2. Escribe en el textarea "Rationale for #N" una justificación de 1-3 frases
   - Si Pass: describe QUÉ verificaste y CÓMO funciona
   - Si Fail: describe QUÉ falla específicamente y POR QUÉ

Para Overall Assessment:
1. Click en el botón 1, 2 o 3:
   - **1** = Falla la mayoría de requisitos (app no carga, errores críticos, features principales rotas)
   - **2** = Parcialmente satisfactorio (funciona lo básico pero bugs o features faltantes)
   - **3** = Satisface completamente todos los requisitos razonables del prompt
2. "Overall Rationale": escribe 3-6 frases detalladas mencionando explícitamente:
   - Qué funciona bien
   - Qué no funciona o tiene bugs
   - Problemas de interacción específicos
   - Calidad general del código y UX
3. "Were the rubrics sufficient?": generalmente "Yes" a menos que detectes algo importante no cubierto
4. Si "No": explica qué aspecto faltó

### FASE 6: SUBMIT
1. Revisa que TODOS los campos estén completos (todos los Pass/Fail marcados, todos los rationales escritos, overall rating seleccionado, overall rationale escrito, rubrics sufficient respondido)
2. Click en el dropdown "In progress" (esquina superior derecha)
3. Selecciona "Mark as complete"

## REGLAS IMPORTANTES

- El Rating debe ser CONSISTENTE con los Pass/Fail:
  - Si todos pasan → generalmente 3
  - Si 1-2 menores fallan → generalmente 2
  - Si features críticas fallan → 1
  
- Los rationales deben ser ESPECÍFICOS, nunca genéricos:
  ✅ "The app renders a 16x12 clickable grid. Clicking cells with a selected item places the emoji correctly. Verified with Oak Tree and Rose Bush items."
  ❌ "Works fine" o "Looks good"

- Siempre prueba INTERACTIVIDAD — es lo que más peso tiene en la evaluación

- Si el Live Preview no carga (service_not_listening o blank):
  1. Primero intenta refrescar la página
  2. Si no funciona, click "Reset" en Captured States
  3. Si sigue sin funcionar, copia el código al setup local y evalúa desde ahí
  4. Menciona en el rationale que tuviste que evaluar localmente

- El código está siempre en UN solo archivo (TypeScript React). No hay múltiples archivos.

- El dropdown del editor dice "typescript" — confirma que es un archivo .tsx

## MAPA DE ELEMENTOS INTERACTIVOS DEL FORMULARIO

Los elementos del formulario Feather siguen este patrón de refs:
- Tab "Live Preview" / Tab "Source Code" — tabs para alternar vista
- Botón "Reset" / "Capture" — en la barra de Captured States
- Cada rúbrica tiene: botón Pass, botón Fail, textarea Rationale
- Overall: botones 1/2/3, textarea Overall Rationale
- Rubrics sufficient: botones "Yes, the rubrics were sufficient" / "No, important aspects were missing"
- Textarea "If no, what was missing?"
- Status dropdown: "In progress" → "Mark as complete"

## EJEMPLO DE RATIONALE BIEN ESCRITO

Rubric: "The app should display a grid of cells representing the garden."
Pass ✅
Rationale: "The app displays a 16-column by 12-row grid of clickable cells. Each cell is visually distinct with borders and hover effects. The grid is properly initialized via useState with GridCell objects containing row/col coordinates. Verified both visually in the preview and in the source code (gridRows=12, gridCols=16)."

Rubric: "Clicking on a cell in the grid should set that grid cell to display the emoji icon of the selected item."
Pass ✅  
Rationale: "Tested by selecting Oak Tree (🌳) and clicking multiple grid cells — each cell correctly displays the tree emoji. The handleCellClick function properly maps over the grid state and updates the matching cell with the selectedItem. Also verified toggle behavior: clicking the same cell again removes the item."

Rubric: "The app should display controls for the name and emoji icon of a new list item."
Fail ❌
Rationale: "The app has a text input for the item name and a text field for the emoji, but there is no actual emoji picker UI. Users must manually type or paste emojis, which doesn't meet the expectation of 'picking' emojis as described in the prompt. The source code confirms only a plain text input with useState('🌸') default — no picker component."
```

---

## RESUMEN DEL WORKFLOW PARA REFERENCIA RÁPIDA

```
PASO 1: Abrir URL de la tarea → Leer Prompt + Rúbricas
PASO 2: Tab Source Code → Copiar código completo
PASO 3: Crear proyecto local (vite react-ts) → Pegar código en App.tsx → npm run dev
PASO 4: Con --chrome, navegar a localhost:5173 → Probar CADA rúbrica interactivamente
PASO 5: Revisar source code (state, handlers, types, edge cases)
PASO 6: Volver a Feather → Marcar Pass/Fail + Rationale para cada rúbrica
PASO 7: Completar Overall Rating (1-3) + Overall Rationale detallado
PASO 8: Responder "Were rubrics sufficient?" 
PASO 9: Dropdown "In progress" → "Mark as complete"
```

## CHECKLIST ANTES DE SUBMIT

```
[ ] ¿Todos los Pass/Fail marcados?
[ ] ¿Todos los rationales escritos (no vacíos)?
[ ] ¿Overall Rating (1-3) seleccionado?
[ ] ¿Overall Rationale escrito (detallado, 3+ frases)?
[ ] ¿"Were rubrics sufficient?" respondido?
[ ] ¿Si "No", explicaste qué faltó?
[ ] ¿Rating es consistente con los Pass/Fail individuales?
[ ] ¿Probaste TODAS las interacciones, no solo la vista visual?
```

## ERRORES COMUNES A EVITAR

1. **No probar interactividad** — Solo mirar la app visualmente sin hacer click. SIEMPRE interactúa.
2. **Rationales genéricos** — "It works" no es aceptable. Describe qué probaste y qué observaste.
3. **Inconsistencia rating vs rubrics** — Si 5/7 rúbricas pasan, el rating no puede ser 1.
4. **Ignorar el source code** — A veces la app SE VE bien pero el código tiene bugs ocultos (ej: estado mutado directamente, effects sin cleanup).
5. **No probar edge cases** — Input vacío, caracteres especiales, muchos items, doble-click rápido.
6. **Olvidar el "Mark as complete"** — Si no cambias el estado, la tarea queda "In progress" y no cuenta.
7. **No verificar que el Live Preview carga** — Si ves `service_not_listening`, NO asumas que la app falla. Intenta refresh/reset primero, luego local.