# Dudas: Pipelines y ColumTransformer
---

## Duda 1: ¿SimpleImputer funciona sobre variables de texto?
---

Sí, funciona perfectamente para texto. Es cierto que su comportamiento por defecto es calcular promedios matemáticos (la media), lo cual solo sirve para números. Sin embargo, al cambiar el parámetro a `strategy='most_frequent'`, el imputer deja de intentar hacer operaciones matemáticas y simplemente cuenta cuál es la palabra o categoría (string) que más se repite en esa columna para usarla como relleno de los valores nulos.


## Duda 2: ¿Cual es la Diferencia entre Pipeline y ColumnTransformer?
---

La diferencia fundamental es la dirección en la que fluyen los datos: en serie (secuencial) vs. en paralelo (enrutamiento).

- **Pipeline (Procesamiento Secuencial)**: Es una línea de ensamblaje recta. El resultado del paso 1 se convierte en la entrada del paso 2.
\
En tu código, el `categorical_transformer` es un Pipeline porque necesitas aplicar dos reglas obligatorias en un orden estricto:
    - primero, el imputer debe tapar los huecos nulos.
    - Segundo, esa data ya limpia se le pasa al `OneHotEncoder` para convertir los textos en matrices de unos y ceros (usando el `handle_unknown='ignore'` para que el programa no colapse si en el futuro aparece una categoría desconocida y simplemente ponga un 0). Si intentaras aplicar el codificador antes de tapar los huecos, el código daría un error.
<p></p>

- **ColumnTransformer (Procesamiento Paralelo)**: Su trabajo no es transformar los datos por sí mismo, sino dividir tu tabla original y que cada parte tenga una ruta de acción.
    - Toma la lista `numerical_cols` y envía esos datos por un carril separado para que apliquen sus reglas de números.
    - Toma la lista `categorical_cols` y envía esos textos por el carril del pipeline categórico.
    
    \
    Ambos procesos ocurren en paralelo y, al finalizar, el `ColumnTransformer` se encarga de pegar los resultados en una sola matriz unificada lista para dársela al modelo.

En resumen: usas un `Pipeline` cuando necesitas encadenar pasos sobre la misma información, y usas un `ColumnTransformer` cuando necesitas aplicar procesos completamente distintos a columnas diferentes al mismo tiempo.


## Duda 3: `'constant'` en SimpleImputer y valor Personalizado
---

Para asignar tu propio valor constante en el `SimpleImputer`, solo necesitas agregar el parámetro `fill_value`. Cuando usas `strategy='constant'` sin declarar nada más, la herramienta pone un **cero por defecto** para números y la palabra **"missing_value" para textos**. Al agregar el parámetro, tú tomas el control total:


```python
# Por defecto 
# Rellena con 0 huecos númericos
# Rellena con "missing_value" huecos de tipo texto
my_imputer = SimpleImputer(strategy='constant')

# Para rellenar huecos numéricos con un valor específico
my_imputer_num = SimpleImputer(strategy='constant', fill_value=-999)

# Para rellenar huecos categóricos con un texto personalizado
my_imputer_txt = SimpleImputer(strategy='constant', fill_value='Sin Registro')
```

## Duda 4: Los steps y transformers pueden tener cualquier ID
---

Los IDs de las tuplas internas (como `'imputer'`, `'onehot'`, `'num'`, `'cat'`),\
**Sí**, pueden ser cualquier palabra inventada (preferiblemente corta y sin espacios).


## Duda 5: Para que sirven los IDs (duda 4) de las tuplas internas a los Pipeline y los ColumTransformers
---

No son simples adornos; estas etiquetas cumplen tres funciones críticas a medida que el proyecto escala:

### Extracción de piezas individuales
---
Un `Pipeline` o un `ColumnTransformer` empaqueta muchas herramientas. Si después de entrenar tu modelo quieres revisar qué categorías exactas aprendió el codificador o qué promedios calculó el imputer, necesitas una forma de *"sacar"* esa pieza específica. Puedes hacerlo usando el ID: `mi_pipeline.named_steps['imputer']`.

### Optimización de hiperparámetros (Tuning)
---
Más adelante, querrás que el programa pruebe automáticamente cientos de combinaciones para ver cuál arroja el mejor resultado. Usarás estos IDs para decirle a la computadora qué tuerca apretar. Por ejemplo, si quieres que pruebe si es mejor usar la media o la mediana, escribirías algo como `imputer__strategy=['mean', 'median']`. El doble guion bajo le indica al sistema: "entra a la pieza etiquetada como `'imputer'` y modifícale su parámetro `'strategy'`".

## Trazabilidad y depuración de errores
---
Si los datos nuevos rompen la secuencia, el mensaje de error de Python no te dirá "falló en el segundo paso". El error imprimirá el ID exacto (ej. "Error en el transformador `'onehot'`"), permitiéndote aislar y corregir la falla de inmediato en lugar de adivinar qué parte de la cadena se rompió.

## Duda 6: ¿Un modelo puede ser un step de Pipeline? y ¿qué más se puede meter ahí?
---

Sí, puedes y debes. Empaquetar el preprocesamiento y el modelo en un solo bloque estandarizado es el objetivo final de usar pipelines. En un `Pipeline` de `Scikit-Learn` puedes meter básicamente dos tipos de piezas, y en un orden muy específico:

- **Transformadores**: Son todas las herramientas que modifican los datos (como `SimpleImputer`, `OneHotEncoder`, escaladores numéricos). Puedes poner tantos como necesites al inicio de la cadena.

- **Estimador (El Modelo)**: Es tu algoritmo predictivo (Random Forest, Regresión Lineal, XGBoost). La regla de oro es que **el modelo solo puede ir al final**, siendo el último paso definitivo del pipeline.


## Duda 7: Uso de `fit` y `pedict` sobre el Pipeline ¿Y si algo no lo tuviese?
---

Al llamar a `my_pipeline.fit(X_train, y_train)`, el pipeline ejecuta una reacción en cadena:

- Toma los datos crudos y aplica internamente `fit_transform` al primer paso (preprocessor en el ejemplo de la clase).

- Toma la tabla de datos ya limpia que salió del paso 1, y se la entrega al paso 2 (el modelo), donde ejecuta un fit tradicional para entrenarlo.

**NOTA**: Para que una herramienta pueda vivir dentro de un pipeline, `Scikit-Learn` exige estrictamente que **todo paso intermedio** posea los métodos `fit` y `transform`.
Si intentas meter un objeto personalizado o una función que no tenga esos métodos, Python arrojará un error de compilación inmediatamente.


## Duda 8: ¿NO hay que llamar al metodo transform?
---
**NO** ¡Esa es exactamente la ventaja principal del pipeline! Encapsula esa complejidad para evitar equivocarse.

Al llamar `my_pipeline.predict(X_valid)`, el pipeline sabe que está en fase de "solo evaluación". De forma automática y silenciosa, toma `X_valid`, le aplica los métodos `transform` del preprocesador (usando las reglas matemáticas que aprendió durante el entrenamiento, garantizando que no haya **data leakage**), y le pasa esa data perfectamente limpia al modelo para que genere las predicciones.


## Duda 9: ¿Qué pasa si se guardara más de un modelo en el pipeline?
---
Un Pipeline estándar de `Scikit-Learn` no soporta múltiples modelos de forma secuencial. La razón es matemática: un transformador (como el imputer) recibe una tabla y devuelve una tabla, pero un modelo recibe una tabla y devuelve una simple matriz de predicciones (una sola columna de resultados).

Si pones un modelo en el paso 2 y otro en el paso 3, el paso 3 recibiría predicciones en lugar de datos para analizar, y el código colapsaría.

Si necesitas usar varios modelos para una misma tarea (una técnica avanzada conocida como **Ensembles** o **Stacking**), el enfoque correcto es agrupar esos modelos usando herramientas específicas (como `VotingRegressor` o `StackingRegressor`) y luego colocar ese "super-modelo agrupado" como el único paso final dentro del pipeline.