# Inicialización de proyecto MkDocs Material

## Requisitos

- Python 3.10 o superior
- Git
- Cuenta de GitHub

## Estructura del proyecto

```text
guia-documentacion/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docs/
│   └── index.md
├── .gitignore
├── mkdocs.yml
└── requirements.txt
```

## Crear el proyecto

```bash
python3 -m venv .venv
source .venv/bin/activate

mkdir -p docs
mkdir -p .github/workflows

touch docs/index.md
touch mkdocs.yml
touch requirements.txt
touch .gitignore
touch .github/workflows/deploy.yml
```

## Dependencias

Contenido de `requirements.txt`:

```txt
mkdocs
mkdocs-material
```

Instalar dependencias:

```bash
pip install -r requirements.txt
```

## Configuración mínima de MkDocs

Contenido de `mkdocs.yml`:

```yaml
site_name: Guía de documentación

theme:
  name: material
  language: es
  features:
    - navigation.sections
    - navigation.top
    - search.suggest
    - content.code.copy

nav:
  - Inicio: index.md
```

## Página inicial

Contenido de `docs/index.md`:

```markdown
# Guía de documentación

Contenido pendiente.
```

## Archivo `.gitignore`

```gitignore
.venv/
site/
__pycache__/
*.pyc
.DS_Store
```

## GitHub Actions

Contenido de `.github/workflows/deploy.yml`:

```yaml
name: Publicar documentación

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Descargar repositorio
        uses: actions/checkout@v4

      - name: Configurar Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Instalar dependencias
        run: pip install -r requirements.txt

      - name: Publicar documentación
        run: mkdocs gh-deploy --force
```

## Ejecutar localmente

```bash
mkdocs serve
```

Abrir en el navegador:

```text
http://127.0.0.1:8000
```

## Inicializar Git

```bash
git init
git branch -M main
git add .
git commit -m "Inicializar proyecto de documentación"
```
