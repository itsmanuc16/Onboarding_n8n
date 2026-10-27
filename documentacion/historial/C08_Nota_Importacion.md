---
tipo: nota-workflow
proyecto: Onboarding
clase: "08"
tags:
  - n8n
  - importacion
---

# C08 — Orden de importación y credenciales

Cada integrante tiene su propia instalación de n8n. Al importar, los IDs de workflow cambian y las credenciales no viajan en el JSON (el JSON solo guarda su nombre y su ID de la instalación de origen). Por eso hay que importar en este orden y volver a seleccionar lo que se indica.

## Archivos

| Archivo                          | Workflow      | Responsabilidad                                                                                                                 |
| -------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `Preparar_Tarea_Onboarding.json` | Subworkflow   | Recibe `id_onboarding`, `nombre`, `area_responsable`, `tipo_contrato`; devuelve `id_onboarding`, `asunto`, `cuerpo`, `id_lista` |
| `Manejador_Errores.json`         | Error Trigger | Crea una tarjeta en `Incidentes` cuando el principal falla                                                                      |
| `C08_Onboarding_Semana8.json`    | Principal     | Webhook, validación, duplicados, llamada al subworkflow, POST a Trello y respuestas                                             |

## Orden de importación

En n8n: workflow nuevo → menú **⋯ → Import from file**, y **guardar** cada uno con su nombre.

1. **`Preparar_Tarea_Onboarding`**. Va primero porque el principal lo llama.
2. **`Manejador_Errores`**. Va antes que el principal porque se asocia en sus Settings.
3. **Principal**.

## Qué se reselecciona después de importar

| Workflow | Nodo o lugar | Qué volver a seleccionar |
| --- | --- | --- |
| `Preparar_Tarea_Onboarding` | — | Nada: no usa credenciales |
| `Manejador_Errores` | `Crear tarjeta en Incidentes` | Credencial **Trello API** |
| Principal | `Webhook nuevo-colaborador` | Credencial **Header Auth** |
| Principal | `GET tarjetas Trello` | Credencial **Trello API** |
| Principal | `Crear tarjeta Trello` | Credencial **Trello API** |
| Principal | `Call 'Preparar_Tarea_Onboarding'` | Workflow → From list → `Preparar_Tarea_Onboarding` (el JSON trae el ID de la instalación de origen) |
| Principal | ⋯ → Settings → Error workflow | `Manejador_Errores` |

## Credenciales que hay que crear (sin escribir sus valores aquí)

| Credencial | Tipo en n8n | Datos que pide | Dónde se obtiene |
| --- | --- | --- | --- |
| `Trello account` | Trello API | API Key y API Token | trello.com/power-ups/admin → app nueva → "Autenticación Trello" → generar API Key → enlace Token |
| `Header Auth account` | Header Auth | Name (`X-Onboarding-Key`) y Value | Se define en el equipo; quien llama al Webhook debe enviar la misma cabecera |

## Publicación

En esta versión de n8n:
- El principal solo se publica si `Preparar_Tarea_Onboarding` está publicado.
- El Error workflow solo se puede asociar si `Manejador_Errores` está publicado.

Ninguno de los dos se ejecuta solo: el subworkflow espera la llamada del principal y el manejador espera un fallo.

El principal se publica solo para las pruebas de producción y se despublica al terminar.

## Datos fijos (no son secretos)

| Dato | Valor |
| --- | --- |
| Tablero Trello | `Onboarding` (`6abb0b3e4752d7ac47ad81a1`) |
| idList TI | `6abb0b3e4752d7ac47ad81a7` |
| idList RRHH | `6abb0b3e4752d7ac47ad81a8` |
| idList Operaciones | `6abb0b3e4752d7ac47ad81a9` |
| idList Incidentes | `6adcf35df033a49cba03f0c0` |

Si se usa un tablero propio, hay que cambiar estos idList en las 3 ramas del subworkflow, en `GET tarjetas Trello` (ID del tablero) y en el manejador.

## Cómo repetir una prueba con la URL de producción (PowerShell)

Con el principal publicado. Reemplazar `<valor>` por el valor de la cabecera, sin guardarlo en archivos.

```powershell
$headers = @{ "X-Onboarding-Key" = "<valor>" }
$body = '{"nombre":"Lucía Ramírez","area":"TI","fecha_ingreso":"2026-11-09","tipo_contrato":"fijo"}'
Invoke-RestMethod -Method Post -Uri "http://localhost:5678/webhook/nuevo-colaborador" -Headers $headers -ContentType "application/json; charset=utf-8" -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
```

- Sin `fecha_ingreso` → 400.
- Repetir el mismo cuerpo → 409.
- Agregar `"forzar_error": true` → falla la ejecución y aparece una tarjeta en `Incidentes`.

La URL `/webhook-test/` solo atiende una llamada después de *Listen for test event* y no activa el manejador.

## Comprobación rápida

- Ningún nodo muestra la alerta de credencial faltante.
- El nodo de subworkflow muestra el nombre `Preparar_Tarea_Onboarding`.
- Settings del principal muestra `Manejador_Errores` como Error workflow.
