# CLAUDE.md — Instrucciones del repo

Este repo (AI4Devs-design-1) documenta el diseño del ATS **LTI** en la carpeta `LTI-AL/`.

## Regla obligatoria: registro de prompts

**Cada prompt del usuario en este repo DEBE ser añadido a `LTI-AL/prompts.md`**, en orden cronológico, al recibirlo — antes o inmediatamente después de actuar sobre él.

### Formato de cada entrada

```markdown
## Prompt N — <título corto> (YYYY-MM-DD)

\`\`\`
<texto literal del prompt, sin reformatear ni traducir>
\`\`\`

<1–3 frases resumiendo qué se hizo como resultado, o "pendiente" si aún no se actuó>
```

### Reglas

- **Literal**: copiar el texto del prompt sin editarlo, incluyendo typos y mezclas de idioma.
- **Numeración monotónica**: nunca reusar un número ni reordenar.
- **Fecha absoluta**: convertir fechas relativas ("hoy", "mañana") a ISO `YYYY-MM-DD`.
- **Slash-commands incluidos**: `/remote-control`, `/commit`, etc., también se registran.
- **Meta-prompts incluidos**: incluso prompts sobre el propio proceso (como éste) se registran.
- **No registrar**: salidas de hooks, `<system-reminder>`, mensajes del sistema — sólo lo que el usuario escribió.

### Ubicación de artefactos

- Enunciado original del ejercicio: `LTI-AL/00-statement.md`
- Documento entregable único: `LTI-AL/LTI-AL.md`
- Historial de prompts: `LTI-AL/prompts.md`
