# PokeAPI + Pokémon Database: pipeline de datos para aprender

Proyecto **didáctico** de ingeniería de datos. Construye, paso a paso y con explicaciones, un dataset limpio de los
**1025 Pokémon** combinando dos fuentes de datos distintas: **web scraping** y una **API REST**.

![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![pandas](https://img.shields.io/badge/pandas-2.x%20%7C%203.x-150458)
![Licencia](https://img.shields.io/badge/licencia-MIT-green)

## Contenido

- [¿Qué aprenderás?](#qué-aprenderás)
- [Arquitectura del pipeline](#arquitectura-del-pipeline)
- [Inicio rápido](#inicio-rápido)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Recorrido del notebook](#recorrido-del-notebook)
- [El dataset resultante](#el-dataset-resultante)
- [Decisiones de diseño](#decisiones-de-diseño-lecciones-para-clase)
- [Configuración](#configuración)
- [Problemas frecuentes](#problemas-frecuentes)
- [Ejercicios](#ejercicios)
- [Buenas prácticas y uso ético](#buenas-prácticas-y-uso-ético)
- [Fuentes y licencia](#fuentes-y-licencia)

## ¿Qué aprenderás?

| Tema | Concepto |
|---|---|
| **Extract** | Web scraping con `requests` + `BeautifulSoup`; consumo de una API REST con JSON |
| **Resiliencia** | Reintentos, *exponential backoff*, manejo de `404` y `429`, `timeout`, caché en disco |
| **Transform** | Grano de una tabla, tipado con `Int64`, joins validados, limpieza, *data quality checks* |
| **Load** | Persistir en CSV con un formato reutilizable |
| **Análisis** | `groupby`, `explode`, `nlargest` y gráficos con `matplotlib` |
| **Ingeniería** | Funciones pequeñas con *type hints* y *docstrings*, constantes de configuración, `logging` |

## Arquitectura del pipeline

```
 pokemondb.net ─ scraping ─▶ df_scrape ─▶ df_base ──┐
 (1261 filas, con formas)    (todo texto)  (1025)   │
                                                    ├─ merge por id ─▶ df_final ─▶ limpieza ─▶ validación ─▶ CSV
 pokeapi.co ──── API REST ─▶ df_api ────────────────┘   (1 a 1)
 (1025 peticiones, con caché)  (1025)
```

| Etapa | Función | Entrada | Salida |
|---|---|---|---|
| 1 | `scrape_pokemondb` | URL | DataFrame con una fila por *forma* (todo como texto) |
| 2 | `cast_numeric_columns` + `select_base_forms` | DataFrame crudo | Una fila por Pokémon, con tipos correctos |
| 3 | `fetch_pokemon_data` / `fetch_all_pokemon` | Lista de IDs | DataFrame con altura, peso, experiencia y habilidades |
| 4 | `merge_datasets` | scraping + API | DataFrame combinado (join 1 a 1 verificado) |
| 5 | `clean_dataset` + `validate_dataset` | DataFrame combinado | Dataset limpio, o error si incumple las reglas |
| 6 | `save_dataset` | Dataset limpio | `data/pokemon_dataset.csv` |

## Inicio rápido

### 1. Clonar e instalar

```bash
git clone https://github.com/porrasgp/PokeAPI---pokemondb.git
cd PokeAPI---pokemondb

python -m venv .venv
source .venv/bin/activate        # En Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Abrir el notebook

```bash
jupyter lab "PokeAPI - pokemondb.ipynb"
```

Ejecuta las celdas en orden (`Run > Run All Cells`).

### 3. Demo rápida para clase

Descargar los 1025 Pokémon tarda unos **3–5 minutos** la primera vez. Para una demostración en vivo, cambia en la
celda de configuración:

```python
MAX_POKEMON = 25      # en lugar de None
```

Con 25 Pokémon el notebook completo se ejecuta en unos 10 segundos.

### Solo quiero los datos

El resultado ya está en [`data/pokemon_dataset.csv`](data/pokemon_dataset.csv):

```python
import pandas as pd

df = pd.read_csv("data/pokemon_dataset.csv")
df["types"] = df["types"].str.split("|")
df["abilities"] = df["abilities"].str.split("|")
```

## Estructura del repositorio

```
.
├── PokeAPI - pokemondb.ipynb   # El notebook de la clase (con explicaciones y salidas)
├── EJERCICIOS.md               # Ejercicios básicos, intermedios y avanzados
├── data/
│   └── pokemon_dataset.csv     # Dataset resultante (1025 filas)
├── requirements.txt            # Dependencias
├── .gitignore                  # Ignora .cache/, entornos virtuales, etc.
├── LICENSE
└── README.md
```

La carpeta `.cache/` se crea sola al ejecutar el notebook y no se sube a Git.

## Recorrido del notebook

| Paso | Qué ocurre | Idea clave |
|---|---|---|
| **0** | Configuración y verificación de dependencias | Una sola zona de constantes; `logging` en lugar de `print` |
| **1** | Scraping de `pokemondb.net/pokedex/all` | Descargar, parsear y recorrer la tabla; extraer sin transformar |
| **2** | Tipar y decidir el *grano* | ¿Qué representa una fila? Una forma vs. un Pokémon |
| **3** | Consumo de la PokeAPI | Llave estable (ID), `Session`, backoff exponencial, caché |
| **4** | Merge de ambas fuentes | `validate="one_to_one"` e `indicator=True` para no confiar a ciegas |
| **5** | Limpieza y validación | Reglas de calidad que hacen fallar el pipeline de forma ruidosa |
| **6** | Guardado en CSV | Nombre de archivo fijo; listas serializadas con `\|` |
| **7** | Resumen por etapas | Cada diferencia de filas tiene explicación |
| **8** | Análisis | Top 10, promedio por tipo y gráfico de barras |

## El dataset resultante

**1025 filas × 14 columnas**, una por Pokémon (forma base).

| Columna | Tipo | Fuente | Descripción |
|---|---|---|---|
| `id` | entero | scraping | Número de Pokédex nacional (1–1025), llave única |
| `name` | texto | scraping | Nombre |
| `types` | lista (`A\|B` en el CSV) | scraping | Uno o dos tipos, por ejemplo `Grass\|Poison` |
| `total` | entero | scraping | Suma de las seis estadísticas base |
| `hp` | entero | scraping | Puntos de salud |
| `attack` | entero | scraping | Ataque |
| `defense` | entero | scraping | Defensa |
| `sp_atk` | entero | scraping | Ataque especial |
| `sp_def` | entero | scraping | Defensa especial |
| `speed` | entero | scraping | Velocidad |
| `height_m` | decimal | API | Altura en **metros** |
| `weight_kg` | decimal | API | Peso en **kilogramos** |
| `base_experience` | entero | API | Experiencia base al derrotarlo |
| `abilities` | lista (`a\|b` en el CSV) | API | Habilidades, por ejemplo `overgrow\|chlorophyll` |

Ejemplo:

| id | name | types | total | hp | attack | defense | sp_atk | sp_def | speed | height_m | weight_kg | base_experience | abilities |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Bulbasaur | Grass\|Poison | 318 | 45 | 49 | 49 | 65 | 65 | 45 | 0.7 | 6.9 | 64 | overgrow\|chlorophyll |
| 4 | Charmander | Fire | 309 | 39 | 52 | 43 | 60 | 50 | 65 | 0.6 | 8.5 | 62 | blaze\|solar-power |

### Reglas de calidad que cumple

El pipeline verifica (y falla si no se cumple alguna) que:

- `id` es único.
- No hay nulos en `id`, `name` ni estadísticas.
- `total` es igual a la suma de las seis estadísticas.
- Altura y peso son positivos.
- Cada Pokémon tiene al menos un tipo.

## Decisiones de diseño (lecciones para clase)

El notebook original funcionaba, pero tenía varios problemas **muy habituales en proyectos reales**. Corregirlos es
parte de la lección:

| Problema original | Consecuencia | Solución actual |
|---|---|---|
| Unir por **nombre** | `Farfetch'd`, `Mr. Mime` o `Nidoran♀` daban `404` porque la API los escribe distinto | Unir por **ID**, una llave estable en ambas fuentes |
| Filas repetidas por forma (Mega, regionales...) | Se pedían ~1200 veces datos de ~1025 Pokémon | `select_base_forms` define el grano y registra cuántas filas se descartan |
| Pérdida **silenciosa** de ~200 filas dentro del `merge` | Nadie sabía por qué 1215 filas se volvían 1025 | Paso explícito + log; `validate="one_to_one"` en el merge |
| Peso y altura desconocidos rellenados con `0` | Un `0` es un dato falso que contamina promedios | Se dejan como `NaN`; la validación exige valores positivos |
| Unidades sin convertir | La API entrega decímetros y hectogramos | `height_m` y `weight_kg` (÷ 10), con la unidad en el nombre |
| Sin caché | Cualquier fallo obliga a repetir todo | Caché JSON en `.cache/` con guardado parcial cada 100 peticiones |
| Un solo reintento con espera fija | Poca resistencia a límites de tasa | *Exponential backoff* (1 s, 2 s, 4 s) |
| Archivo de salida con `uuid` | Cada ejecución crea un archivo nuevo | Nombre fijo `data/pokemon_dataset.csv` |
| Código del guardado duplicado y sin validaciones | Confusión y errores silenciosos | Una sola función y `validate_dataset` |

## Configuración

Todo se controla desde la celda del **Paso 0**:

| Constante | Valor por defecto | Para qué sirve |
|---|---|---|
| `URL_POKEMONDB` | `https://pokemondb.net/pokedex/all` | Página que se scrapea |
| `URL_POKEAPI` | `https://pokeapi.co/api/v2/pokemon/{id}` | Plantilla del endpoint de la API |
| `MAX_POKEMON` | `None` | Límite de Pokémon (`None` = todos). Útil para demos |
| `PAUSA_ENTRE_PETICIONES` | `0.1` | Segundos de espera entre llamadas a la API |
| `CACHE_PATH` | `.cache/pokeapi_cache.json` | Caché de respuestas de la API |
| `OUTPUT_PATH` | `data/pokemon_dataset.csv` | Archivo de salida |

Para empezar desde cero (volver a pedir todo a la API), borra la caché:

```bash
rm -rf .cache
```

## Problemas frecuentes

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `ValueError: No se encontró la tabla con id='pokedex'` | La página cambió su HTML | Inspecciona la página (F12) y ajusta el selector en `scrape_pokemondb` |
| `requests.exceptions.ConnectionError` | Sin internet o sitio caído | Revisa tu conexión; la caché conserva lo ya descargado |
| Mensajes `Límite de tasa (429)` | Demasiadas peticiones seguidas | Sube `PAUSA_ENTRE_PETICIONES`; el *backoff* se encarga del resto |
| `AssertionError: Reglas de calidad incumplidas` | Un dato no cumple una regla | Mira en el log cuál regla falló (`[FALLA]`) y revisa esas filas |
| `ModuleNotFoundError` | Faltan dependencias | `pip install -r requirements.txt` |
| La ejecución completa es lenta | Son más de 1000 peticiones | Usa `MAX_POKEMON = 25` o espera: la segunda vez usa la caché |
| El número de filas del scraping cambia (`1261`, etc.) | El sitio agrega nuevas formas | Es normal; el resultado final sigue siendo una fila por Pokémon |

## Ejercicios

En [`EJERCICIOS.md`](EJERCICIOS.md) hay **12 ejercicios** graduados y preguntas de discusión:

- **Básico:** consultas con `pandas` (más pesado, un tipo vs. dos tipos, filtrar por habilidad).
- **Intermedio:** columnas derivadas, `pd.cut`, nuevas reglas de calidad.
- **Avanzado:** incluir formas alternativas, nuevos endpoints, descargas en paralelo y convertir el notebook en un
  módulo con `argparse` y `pytest`.

## Buenas prácticas y uso ético

- **Sé amable con los servidores.** El notebook espera entre peticiones y usa caché. No elimines esas protecciones ni
  lances cientos de hilos contra servicios gratuitos.
- **Revisa los términos de uso** de cualquier sitio antes de hacer scraping. Prefiere siempre una API oficial cuando
  exista.
- **Identifícate.** El notebook envía un `User-Agent` descriptivo.
- **No dependas de un scraper para producción.** Si el HTML cambia, se rompe: por eso las validaciones fallan de forma
  clara en vez de producir datos erróneos.
- Este proyecto es **exclusivamente educativo**.

## Fuentes y licencia

- Estadísticas base: [Pokémon Database](https://pokemondb.net/pokedex/all)
- Peso, altura y habilidades: [PokeAPI](https://pokeapi.co/)
- Pokémon y sus nombres son marcas de Nintendo, Game Freak y The Pokémon Company. Este repositorio no está afiliado
  a ellas.

El código se distribuye bajo la licencia incluida en [`LICENSE`](LICENSE).
