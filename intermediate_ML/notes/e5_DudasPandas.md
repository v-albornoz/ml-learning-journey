# Uso de get_dummies y align
---

## Código de Referencia
---
```python
# One-hot encode the data (to shorten the code, we use pandas)
X_train = pd.get_dummies(X_train)
X_valid = pd.get_dummies(X_valid)
X_test = pd.get_dummies(X_test)

X_train, X_valid = X_train.align(X_valid, join='left', axis=1)
X_train, X_test = X_train.align(X_test, join='left', axis=1)
```

## ¿Qué hace `get_dummies` de Pandas?
---
`pd.get_dummies()` aplica **One-Hot Encoding** a las columnas categóricas (`string` o `category`) de un DataFrame: convierte cada categoría distinta en una nueva columna binaria con valores `1` (o `True`, según la versión de pandas) si la fila pertenece a esa categoría y `0` (o `False`) si no.

Las columnas numéricas que ya existían se mantienen tal cual.

Ejemplo rápido
Supón que tienes una columna `Color` con valores `["Rojo", "Azul", "Rojo"]`:

| Color | Color_Azul | Color_Rojo |
| --- | --- | --- |
| Rojo | 0 | 1 |
| Azul | 1 | 0 |
| Rojo | 0 | 1 |


## ¿Por qué se usa junto con `.align(..., join='left', axis=1)`?
Cuando codificas por separado `X_train`, `X_valid` y `X_test`, pueden surgir desajustes en las columnas:

- Una categoría aparece en `X_train`, pero no en `X_valid` o `X_test`: `get_dummies` no creará esa columna en validación/prueba, pero el modelo entrenado en `X_train` sí la espera.

- Una categoría nueva aparece solo en `X_valid` o `X_test`: `get_dummies` creará una columna extra que no existía en `X_train`.

Al hacer:
```python
X_train, X_valid = X_train.align(X_valid, join='left', axis=1)
```

- `axis=1` indica que la alineación es sobre las columnas.

- join='left' fuerza a que `X_valid` (y luego `X_test`) tenga exactamente las mismas columnas que `X_train`:

    - Si falta una columna que sí estaba en `X_train`, la agrega y rellena con `NaN` (en versiones recientes de pandas a veces conviene hacer `.fillna(0)` después).

    - Si sobra una columna que no estaba en `X_train`, la descarta.


## Detalles prácticos a tener en cuenta
---
- **Tipos de datos:** En versiones recientes de pandas, `get_dummies()` suele devolver booleanos (`True`/`False`) en lugar de `0`/`1`, a menos que pases `dtype=int`.

- **Valores faltantes (`NaN`):** Por defecto `dummy_na=False` (ignora los `NaN`). Si quieres una columna específica para valores nulos, usa `dummy_na=True`.

- **Trampa de multicolinealidad:** Si usas modelos lineales o regresión logística, a veces se pasa `drop_first=True` para eliminar una columna redundante por cada variable categórica; en modelos basados en árboles (**Random Forest**, **XGBoost**) suele dejarse `drop_first=False`.