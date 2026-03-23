# Pasos para usar el PR Review Action en tu repositorio

Este documento describe cómo configurar el action en un repositorio cliente (por ejemplo, `JurojinPoker/jurojin`).

## Requisitos previos

1. **Secret `OPENAI_API_KEY`** en tu repo (Settings → Secrets and variables → Actions).
2. El action publicado en GitHub (p.ej. `JurojinPoker/jurojinpoker-pr-review-action`).

---

## Paso 1: Crear el workflow

Crea el archivo `.github/workflows/pr-review.yml` en tu repositorio (o el nombre que prefieras):

```yaml
name: PR Summary (Jurojin)

on:
  pull_request:
    branches: [master, staging]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  openai_pr_review:
    runs-on: ubuntu-latest
    name: Jurojin PR Review
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Review PR with OpenAI
        uses: JurojinPoker/jurojinpoker-pr-review-action@v1
        with:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ github.token }}
          REPOSITORY: ${{ github.repository }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          GPT_MODEL: gpt-4o-mini
          MAX_TOKENS: "1500"
          prompt_style: team
```

---

## Paso 2: Elegir el prompt style

**No necesitas añadir archivos de prompts.** El action incluye tres estilos:

| Style     | Uso                                  |
|-----------|--------------------------------------|
| `team`    | Resumen interno para el equipo       |
| `technical` | Revisión técnica de código        |
| `users`   | Resumen orientado a usuarios/producto |

Puedes usar:

- Un estilo fijo: `prompt_style: team`
- O seleccionar por rama (como en tu configuración actual):

```yaml
      - name: Select prompt style
        id: select_style
        env:
          BASE_REF: ${{ github.base_ref }}
          OVERRIDE_STYLE: ${{ vars.PR_REVIEW_STYLE }}
        run: |
          set -e
          if [ -n "$OVERRIDE_STYLE" ]; then
            STYLE="$OVERRIDE_STYLE"
          else
            case "$BASE_REF" in
              staging) STYLE=technical ;;
              master) STYLE=team ;;
              *) STYLE=team ;;
            esac
          fi
          echo "Using OpenAI prompt style: ${STYLE}"
          echo "prompt_style=${STYLE}" >> "$GITHUB_OUTPUT"

      - name: Review PR with OpenAI
        uses: JurojinPoker/jurojinpoker-pr-review-action@v1
        with:
          # ... otros inputs ...
          prompt_style: ${{ steps.select_style.outputs.prompt_style }}
```

---

## Paso 3: (Opcional) Override con archivo custom

Si quieres usar tus propias instrucciones en lugar de los estilos predefinidos:

1. Crea un archivo en tu repo, por ejemplo `.github/pr-review-prompts/mi-prompt.txt`.
2. Configura el action con `prompt_file`:

```yaml
with:
  # ... otros inputs ...
  prompt_file: .github/pr-review-prompts/mi-prompt.txt
```

`prompt_file` tiene prioridad sobre `prompt_style`. Solo usa uno u otro.

---

## Resumen de cambios para JurojinPoker/jurojin

Con la nueva versión del action:

1. **No necesitas** crear `.github/pr-review-prompts/` en jurojin.
2. Tu workflow actual ya está bien: usa `prompt_style: team` (o el que venga de `select_style`).
3. **Actualiza la referencia al action** a la versión que incluya estos cambios (por ejemplo, publica una nueva release `v1` o `v2` del action y apunta a ella).

---

## Troubleshooting

| Error | Causa | Solución |
|-------|-------|----------|
| `prompt style file not found` | Estás usando una versión antigua del action que busca archivos en tu repo | Actualiza a la versión que usa prompts integrados |
| `prompt_file not found` | Usas `prompt_file` pero el archivo no existe en tu repo | Crea el archivo o quita `prompt_file` para usar `prompt_style` |
| ` Missing required environment variables` | Faltan secrets/inputs | Verifica que `OPENAI_API_KEY` y `GITHUB_TOKEN` estén configurados |
