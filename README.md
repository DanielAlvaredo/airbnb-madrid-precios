# Precios de pisos turísticos en Madrid (Airbnb)

**Pregunta de negocio:** ¿qué precio debería fijar un propietario para un alojamiento nuevo en Madrid y qué factores determinan el precio?

Proyecto de análisis de datos de principio a fin: limpieza con Python y SQL, regresión hedónica (econometría), modelo predictivo (Random Forest) y dashboard interactivo con un simulador de precio.

**Dashboard interactivo:** [Ver en Tableau Public](https://public.tableau.com/app/profile/daniel.alvaredo.l.pez/viz/PreciosdepisostursticosenMadridAirbnb/PreciosdepisostursticosenMadrid)



!\[Captura del dashboard](img/dashboard.png)



## Resultados principales

1. **La capacidad es el principal motor del precio.** Cada persona adicional de capacidad se asocia con un precio en torno a un **15 % mayor**, manteniendo lo demás constante.
2. **El tipo de alojamiento pesa mucho.** Frente a un piso completo, una habitación privada cuesta aproximadamente un **34 %** menos y una habitación compartida un **73 %** menos (con barrio controlado).
3. **El efecto superhost es pequeño.** En bruto, los superhosts cobran un 7,7 % más. Al controlar por tamaño, tipo y barrio, la diferencia baja a un **5,5 %** aproximadamente.
4. **La ubicación importa, pero explica menos de lo esperado.** Añadir el barrio sube el R² ajustado de la regresión de 0,422 a 0,492.
5. **Un Random Forest predice mejor que la regresión lineal**, pero con un error medio de unos **62 € por noche** (ver limitaciones).
6. **Centro caro, periferia barata.** Entre los barrios con al menos 30 pisos completos, los de mayor mediana de precio son Jerónimos, Recoletos, Sol, Goya y Cortes.

## Datos

* **Fuente:** [Inside Airbnb](https://insideairbnb.com/get-the-data/), Madrid, archivo `listings.csv.gz` (listados detallados). Fecha de la descarga: 2026-09-22. Consulta las condiciones de uso de los datos en la web de Inside Airbnb.
* **Tamaño inicial:** 23.109 alojamientos y 90 columnas.
* **Tras la limpieza:** 18.573 alojamientos.

## Metodología

### 1\. Limpieza (Python, pandas)

* El precio venía como texto con el símbolo `$`; se convirtió a número.
* Se eliminó el **18,7 %** de alojamientos sin precio (4.324 filas), porque el precio es la variable a explicar y no se imputó.
* Se descartaron los precios extremos, fuera del rango **20-1.000 € por noche** (212 filas). El máximo original era de 36.883 € y la mediana de 151 €.
* Las valoraciones ausentes (2.613 alojamientos) corresponden en el 100 % de los casos a alojamientos **sin reseñas**; no son datos perdidos. En lugar de imputarlas, se creó la variable `has\_reviews`.
* Se creó `log\_price` para trabajar con el logaritmo del precio, que tiene una distribución mucho más simétrica y permite interpretar los coeficientes como variaciones porcentuales.

### 2\. Base de datos y SQL

Los datos limpios se guardaron en una base de datos **SQLite** y se exploraron con consultas SQL (agregaciones por barrio, tipo de alojamiento y superhost). Para evitar medias poco fiables, las consultas por barrio exigen un mínimo de alojamientos (`HAVING COUNT(\*) >= 50`).

### 3\. Regresión hedónica (statsmodels)

Regresión por MCO sobre `log\_price` con errores estándar robustos a heterocedasticidad (HC3).

|Variable|Modelo 1|Modelo 2 (con barrio)|
|-|-|-|
|Habitación privada (vs. piso completo)|-38,2 %|-34,3 %|
|Habitación compartida (vs. piso completo)|-74,1 %|-73,3 %|
|Cada persona más de capacidad|+15,8 %|+15,4 %|
|Superhost|+4,9 %|+5,5 %|
|Tiene reseñas|+7,0 %|+7,7 %|
|R² / R² ajustado|0,422 / 0,422|0,496 / 0,492|

(Efecto porcentual calculado como `exp(coeficiente) - 1`. 18.534 observaciones tras excluir alojamientos sin dormitorios informados o sin dato de superhost y los 29 "Hotel room".)

### 4\. Modelos predictivos (scikit-learn)

División 80 % entrenamiento y 20 % prueba. La configuración del Random Forest se eligió por validación cruzada sobre el conjunto de entrenamiento, y el de prueba se usó solo para la evaluación final.

|Modelo|R² en prueba|Error medio absoluto|
|-|-|-|
|Regresión lineal (con reseñas, superhost y disponibilidad)|0,483|66,6 €|
|Random Forest (con reseñas, superhost y disponibilidad)|0,611|58,4 €|
|Random Forest sin disponibilidad|0,565|61,1 €|
|**Random Forest para alojamiento nuevo (modelo final)**|**0,553**|**62,2 €**|

### 5\. Decisiones de diseño y su justificación

* **Variables excluidas del modelo final.** `availability\_365`, `has\_reviews` y `superhost` se quitaron del modelo que recomienda precios: un propietario que va a publicar un alojamiento nuevo no conoce su disponibilidad futura, no tiene reseñas ni estatus de superhost. Incluirlas habría usado información que no existe en el momento de decidir. Esas variables se mantienen en la regresión, donde sirven para *explicar* el precio. Excluirlas costó poco: el R² de prueba pasó de 0,565 a 0,553.
* **Una incoherencia detectada y corregida.** Una primera versión del simulador mantenía `has\_reviews` y daba predicciones inestables según el valor elegido (en una misma combinación, el precio saltaba más de un 50 % entre "con reseñas" y "sin reseñas"). Comprobar los recuentos mostró que había solo 13-15 alojamientos sin reseñas en esas celdas.
* **Rango de incertidumbre.** El simulador no da un precio único, sino un rango (mínimo-recomendado-máximo) construido con los percentiles 10 y 90 de los errores en el conjunto de prueba. La cobertura se mantiene entre el 77 % y el 82 % en los diez deciles de precio predicho.
* **Umbral de 30 pisos completos por barrio** en el dashboard, para evitar medianas poco fiables (de 126 barrios a 71).

## Dashboard (Tableau Public)

El dashboard tiene tres partes:

1. **Precio mediano por barrio**, filtrable por tipo de alojamiento.
2. **Mapa de burbujas por barrio**: el color indica la mediana de precio y el tamaño, el número de alojamientos.
3. **Simulador de precio**: se elige barrio, tipo de alojamiento, capacidad y dormitorios, y devuelve el precio mínimo, recomendado y máximo, junto con el número de alojamientos en que se apoya (indicador de fiabilidad). Solo incluye combinaciones con al menos 5 alojamientos en los datos.

## Limitaciones

* **El error es grande.** El error medio de unos 62 € sobre una mediana de 151 € es del orden del 40 %. Faltan variables que pesan en el precio real: calidad y estado del piso, fotos, vistas, planta, reformas.
* **Ligera sobrestimación en los precios altos.** En los deciles 8 y 9 de precio predicho, la mediana predicha supera a la real en un 5-9 %.
* **Combinaciones poco frecuentes.** En celdas con pocos alojamientos las predicciones son menos fiables y pueden ser incoherentes (por ejemplo, un precio recomendado que baja al subir la capacidad).
* **Asociaciones, no causalidad.** Tener reseñas o ser superhost se asocia con precios más altos, pero eso no significa que conseguir una reseña o la etiqueta suba el precio.
* **Calidad de datos.** Algunas habitaciones privadas aparecen con 11 o 18 dormitorios, casi seguro errores de registro.
* **Mapa aproximado.** Cada burbuja se sitúa en la posición media de los alojamientos del barrio, no en sus límites reales.
* **Precios de oferta, no de transacción.** El precio es el publicado por el anfitrión, no necesariamente el que paga el huésped.
* **Una sola fotografía en el tiempo.** Los datos son de una fecha concreta, sin considerar la estacionalidad.

## Cómo reproducirlo

1. Instala [Anaconda](https://www.anaconda.com/) (incluye Python, Jupyter, pandas, statsmodels y scikit-learn).
2. Descarga `listings.csv.gz` de Madrid desde [Inside Airbnb](https://insideairbnb.com/get-the-data/) y guárdalo en `data/`. El archivo no se incluye en este repositorio por su tamaño.
3. Ejecuta el notebook `01\_carga\_y\_limpieza.ipynb` de principio a fin.
4. Para el dashboard, abre `data/airbnb\_madrid.xlsx` en Tableau (hojas `listings` y `simulador`).

## Estructura del repositorio

```
airbnb-madrid/
├── 01\_carga\_y\_limpieza.ipynb   # limpieza, SQL, regresión y modelos
├── data/
│   ├── listings\_limpio.csv     # datos limpios
│   ├── simulador\_precios.csv   # tabla del simulador
│   └── airbnb\_madrid.xlsx      # entrada del dashboard
├── img/
│   └── dashboard.png           # captura del dashboard
└── README.md
```

## Herramientas

Python (pandas, NumPy, matplotlib, statsmodels, scikit-learn), SQLite, Jupyter y Tableau Public.

## Autor

Daniel Alvaredo López, estudiante de Economía. [Perfil de Tableau Public](https://public.tableau.com/app/profile/daniel.alvaredo.l.pez)

