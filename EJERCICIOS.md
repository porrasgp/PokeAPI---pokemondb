# Ejercicios

Ejercicios para extender el pipeline de `PokeAPI - pokemondb.ipynb`. Están ordenados de menor a mayor dificultad.
Haz cada uno en una celda nueva al final del notebook (o en una copia del archivo).

> **Consejo:** mientras desarrollas usa `MAX_POKEMON = 25` en la celda de configuración. Gracias a la caché, la
> segunda ejecución con el mismo conjunto no vuelve a llamar a la API.

---

## Nivel básico (explorar y consultar)

### 1. ¿Quién es el más pesado y quién el más liviano?
Usa `idxmax()` / `idxmin()` o `nlargest` / `nsmallest` para mostrar los 5 Pokémon más pesados y los 5 más livianos.
Muestra `name`, `weight_kg` y `height_m`.

<details>
<summary>Pista</summary>

`df_clean.nlargest(5, "weight_kg")[["name", "weight_kg", "height_m"]]`
</details>

### 2. Pokémon de un solo tipo
¿Cuántos Pokémon tienen exactamente **un** tipo y cuántos tienen **dos**?

<details>
<summary>Pista</summary>

`df_clean["types"].str.len().value_counts()`
</details>

### 3. Filtra por habilidad
Lista los Pokémon que tienen la habilidad `levitate`.

<details>
<summary>Pista</summary>

`df_clean[df_clean["abilities"].apply(lambda lista: "levitate" in lista)]`
</details>

### 4. El mejor de cada estadística
Para cada una de las 6 estadísticas (`hp`, `attack`, ...), encuentra el Pokémon con el valor más alto.

---

## Nivel intermedio (transformar)

### 5. Índice de masa corporal (IMC) Pokémon
Crea la columna `bmi = weight_kg / height_m ** 2`. ¿Qué Pokémon tiene el IMC más alto? ¿Y el más bajo?
Después comprueba que no hay divisiones por cero ni valores infinitos.

### 6. Pokémon con tipo principal y secundario
Hoy `types` es una lista. Crea dos columnas, `type_1` y `type_2` (esta última vacía si solo tiene un tipo).
¿Cuál es la combinación de tipos más común?

<details>
<summary>Pista</summary>

`df_clean["types"].str[0]` y `df_clean["types"].str[1]` extraen el primer y segundo elemento de cada lista.
</details>

### 7. Categoriza el poder
Con `pd.cut`, crea la columna `power_tier` con las categorías `bajo` (< 350), `medio` (350–499), `alto` (500–599) y
`legendario` (>= 600). Cuenta cuántos Pokémon hay en cada categoría.

### 8. Una regla de calidad nueva
Agrega una regla a `validate_dataset` que verifique que **ninguna estadística individual** (`hp`, `attack`, ...) sea
mayor que 255. Después rompe los datos a propósito (por ejemplo `df_clean.loc[0, "hp"] = 999`) y comprueba que el
pipeline falla con un mensaje claro.

---

## Nivel avanzado (ampliar el pipeline)

### 9. Incluye las formas alternativas
Hoy el pipeline se queda solo con la forma base (`select_base_forms`). Cambia el diseño para conservar también las
formas (Mega, regionales...) con un grano distinto: **una fila por (id, forma)**.

Preguntas guía:
- ¿Cuál sería ahora la llave del `merge`?
- ¿Cómo se llama una forma en la PokeAPI? Por ejemplo `venusaur-mega` (revisa <https://pokeapi.co/api/v2/pokemon/venusaur-mega>).
- ¿Qué regla de `validate_dataset` deja de tener sentido y cuál debes poner en su lugar?

### 10. Añade el color y la generación
La PokeAPI tiene el endpoint `https://pokeapi.co/api/v2/pokemon-species/{id}` con los campos `color` y `generation`.
Crea una función `fetch_species_data` con la misma estructura (reintentos, backoff, caché) y une el resultado al dataset.
Después grafica cuántos Pokémon hay por generación.

### 11. Descarga en paralelo
Hacer 1025 peticiones en secuencia tarda minutos. Reescribe `fetch_all_pokemon` con
`concurrent.futures.ThreadPoolExecutor` (por ejemplo con 5 hilos). Mide el tiempo antes y después.

**Piensa:** ¿por qué usar demasiados hilos puede ser contraproducente con una API pública? ¿Qué pasa con el código `429`?

### 12. Convierte el notebook en un script
Mueve las funciones a un módulo `pokemon_pipeline.py` y crea una función `main()` que ejecute todos los pasos.
Agrega argumentos de línea de comandos con `argparse` (`--max-pokemon`, `--output`). Escribe una prueba con
`pytest` para `select_base_forms` usando un DataFrame pequeño hecho a mano.

---

## Preguntas de discusión

1. ¿Por qué es mejor unir por `id` que por `name`? ¿Qué otras llaves frágiles has visto (correos, direcciones, fechas)?
2. ¿Qué riesgos tiene el *web scraping* frente a usar una API? (cambios en el HTML, términos de uso, límite de peticiones).
3. En la versión original, se perdían ~200 filas sin ningún aviso. ¿Qué mecanismos del pipeline actual lo evitan?
4. ¿Por qué `peso = 0` es peor que `peso = NaN` cuando hay datos faltantes?
5. ¿Qué pasaría si PokeAPI estuviera caída? ¿Qué parte del pipeline te protege?
