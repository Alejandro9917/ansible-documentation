# Documentacion de Ansible

Repositorio de documentacion para la guia y capacitacion de Ansible, construido con MkDocs Material.

## Requisitos

- Python 3.10 o superior
- Git
- pip

## Inicializar el entorno local

Crear y activar un entorno virtual:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Instalar las dependencias del proyecto:

```bash
pip install -r requirements.txt
```

## Ejecutar la documentacion localmente

Levantar el servidor de desarrollo:

```bash
mkdocs serve
```

Abrir en el navegador:

```text
http://127.0.0.1:8000
```

## Construir el sitio

Generar los archivos estaticos:

```bash
mkdocs build
```

El sitio generado quedara en el directorio `site/`.

## Estructura principal

```text
.
├── docs/
│   └── index.md
├── .github/
│   └── workflows/
│       └── deploy.yml
├── mkdocs.yml
├── requirements.txt
└── README.md
```

## Publicacion

El workflow `.github/workflows/deploy.yml` publica la documentacion en GitHub Pages cuando se hace push a la rama `main`.

Para publicar manualmente desde local, con las dependencias instaladas:

```bash
mkdocs gh-deploy --force
```
