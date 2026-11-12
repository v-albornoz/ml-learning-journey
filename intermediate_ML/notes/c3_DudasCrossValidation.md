# Dudas: Cross Validation - cross_val_score
---

## Duda 1: ¿Por que usar negative MAE? y ¿Cuando Usarla?
---

- **La convención de Scikit-Learn:** Por regla general, todas las métricas de evaluación en Scikit-Learn están diseñadas bajo una lógica: *"a mayor número, mejor es el modelo"* (como la precisión o el $R^2$, donde ganar un 0.95 es mejor que un 0.50).

- **El problema del MAE:** El MAE mide error. Por lo tanto, un MAE bajo (como 5,000) es excelente y un MAE alto (como 50,000) es pésimo. ¡Va al revés de la regla general!

- **La solución (el truco matemático):** Para que todas sus funciones internas funcionen con la misma lógica de *"más alto es mejor"*, Scikit-Learn decidió multiplicar el MAE por -1. De esta forma: Un error de $21,000 se convierte en -21000. Un error de $15,000 se convierte en -15000. Matemáticamente hablando -15000 es mayor que -21000. Así, la librería contenta su regla de que *"un número más alto es mejor modelo"*.

### ¿Cuándo usarla?
Solo se usa exclusivamente dentro de funciones de Scikit-Learn que evalúan bloques completos (como cross_val_score) y que piden un parámetro llamado `scoring`.


## Duda 2: Los ejemplos usaron cross_val_score pero ¿Cuando entrenó?
---

Al ejecutar cross_val_score ocurren tres pasos por cada fold (división):

- **Divide los datos:** por ejemplo para cv=5, corta el dataset en 5 partes iguales.

- **Entrena automáticamente:** Toma 4 partes para entrenar el modelo (fit) y reserva 1 parte para evaluarlo. Luego repite esto 5 veces rotando el pedazo de evaluación.

- **Calcula el error:** En cada una de las 5 rondas, le pasa los datos de prueba al modelo entrenado, calcula el MAE y al final devuelve una lista con 5 puntajes de error diferentes.

En resumen: Esa sola linea de codigo entrena y evalúa 5 veces seguidas. No es necesario escribir código de entrenamiento manual (fit) para cada división; la función lo hace sola.

## Duda 3: ¿Qué hago con los modelos que ya entrene? ¿donde estan?
---
Aquí hay una confusión muy común: `cross_val_score` NO guarda un modelo listo para producción.
Entrena modelos temporales, mide su error y luego los desecha. Su único propósito es dar una estimación confiable de la métrica (el MAE **promedio**).

## Duda 4: ¿Cual es la utilidad real de cross_val_score?
---
`cross_val_score` es una herramienta de validación profesional.

### Referencia del promedio
---
Sin ella para saber si un modelo es bueno, es necesario dividir los datos **una sola vez** a mano (`train_test_split`), entrenar una vez, y asumir que esa única división fuese representativa. Por el contrario con `cross_val_score` no se confia en una sola división, se tiene un promedio de n experimentos, una estimación mucho más robusta de la calidad de un modelo.

### Valores individuales
---
Sirven para medir consistencia y estabilidad del modelo.

- **Uso 1: Detectar Varianza o Inestabilidad (Sensibilidad a los datos)**
Por ejemplo:
    - En el Fold 2, el error fue de $303,164.
    - En el Fold 4, el error bajó a $236,061.
    <p></p>
    Diferencia: ¡Hay casi $67,000 dólares de diferencia en error entre una prueba y otra! Un modelo ideal tendría 5 números muy parecidos entre sí (ejemplo: [270k, 275k, 272k, 269k, 271k]).

    Ver la lista de los 5 scores alerta de algo vital: **El modelo es muy sensible a qué datos le tocan**. En algunas partes del dataset predice razonablemente bien, pero en otras se equivoca feo. 

- **Uso 2: Detectar un Fold Atípico (Outlier)**
Si un solo Fold tiene un valor muy disparejo respecto a los demas (Muy alto o muy bajo), representa un indicativo rapido de que en alguna parte existe un valores atipicos que se requiere limpiar.

En resumen: La lista de valores sirve para decir "Mi modelo es estable y confiable" o "Mi modelo es inestable porque varía mucho de un fold a otro".


## Duda 5: Si los (5) valores resultantes son muy disparejos (alta varianza), ¿qué hago?
---
En ese caso el modelo te está diciendo: "Soy muy inestable y dependiente de los datos con los que me entrenas". Para corregirlo, tienes tres caminos principales:

### 1. (Tuning) Regularizar el modelo / Reducir la complejidad:
---
- En un Random Forest, puedes subir el valor de min_samples_leaf (ej. de 1 a 10 o 20). Esto obliga a los árboles a no hacer reglas tan específicas y suaviza las predicciones.

- O puedes limitar la profundidad máxima (max_depth=10 o 15) para evitar que memorice casos raros de un fold específico.

### 2. (más limpieza) Revisar si hay Outliers o datos sucios:
---
Un par de casas con precios absurdamente altos en un solo fold pueden disparar el error de esa prueba.

### 3. (Avanzado) Usar Stratified K-Fold / Cambiar el tipo de división:
---
Si es un problema de clasificación o los precios están muy sesgados, la división aleatoria simple puede dejar datos difíciles en un solo fold. (Para ejercicios avanzados la funcion `KFold` o `StratifiedKFold` permiten explorar la divicion e identificar que registros cayeron en ese fold)

### Duda 6: ¿Qué hacer si `cross_val_score` tarda mucho?
---
- **Reduce la cantidad de folds:** En lugar de `cv=10`, usa `cv=3` o `cv=5` durante la fase de exploración.

- **Usar el procesamiento en paralelo:** Añade el parámetro `n_jobs=-1` a la función `cross_val_score` (ej: cross_val_score(my_pipeline, X, y, cv=5, n_jobs=-1)). Esto le dice a Scikit-Learn que use todos los núcleos del procesador al mismo tiempo para entrenar los folds en paralelo.


### Duda 7: ¿Qué valor elegir para `cv`?
---
Depende de la cantidad de datos. En general un número más grande permite mayor variabilidad, lo que es especialmente util para datasets pequeños, sin embargo esto tambien aumenta el tiempo de computo.

Recomendaciones:
- `cv=5` es el estandar de la industria. se usa el 90% de las veces. Ofrece un excelente equilibrio entre buena estimación del error y tiempo de cómputo razonable.

- `cv=10` muy común para datasets pequeños a medianos. da una estimación ligeramente más precisa y con menos sesgo, pero requiere el doble de tiempo de cómputo que `cv=5`.

- `cv=3` o `2` (Datasets Gigantes / Modelos pesados). Si entrenar un solo modelo tarda horas (como en redes neuronales o datasets de millones de filas), hacer 5 o 10 iteraciones es inviable. Se reduce el número de folds o directamente se vuelve a un `train_test_split` tradicional de una sola división.

## Notas Finales sobre `cross_val_score`
---
- Sirve única y exclusivamente para evaluar y comparar.

- No guarda ningún modelo final listo para producción. Los modelos temporales que entrena se descartan al terminar.

- Protege de usar una usa división:
    - Si un split deja valores muy raros en la prueba, el modelo pareserá **pésimo**.
    - Si por el contrario deja valores ideales, el modelo parecerá perfecto y **fracasará en producción**.
    <p></p>
    Por lo tanto `cross_val_score` proteje contra ambos sesgos.

## Fujo Real de Diseño
---
Una vez que usas la validación cruzada para tomar todas tus decisiones de diseño (hiperparámetros, limpieza, tipo de modelo) y estás 100% seguro de que el promedio es óptimo, el último paso obligatorio en MLOps es entrenar con el 100% de los datos limpios:

```Python
# 1. Usas cross_val_score solo para experimentar y ajustar decisiones
scores = cross_val_score(my_pipeline, X, y, cv=5, scoring='neg_mean_absolute_error')
print("MAE Promedio:", -1 * scores.mean())

# 2. Una vez aprobado el diseño, entrenas EL MODELO FINAL con TODO tu dataset disponible
my_pipeline.fit(X, y) 

# ¡Listo! Ahora guardas este pipeline final (ej. con joblib o pickle) para desplegarlo en tu app.
```