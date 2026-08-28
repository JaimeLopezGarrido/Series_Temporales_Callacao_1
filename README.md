# Análisis de series temporales en la cuenca del Salar de Callacao

Trabajo práctico de la materia Análisis de Series Temporales (Maestría en Ciencia de Datos,
Universidad Austral). El objetivo fue modelar y pronosticar tres series ambientales de una
cuenca endorreica de la Puna salteña usando la metodología Box-Jenkins, y después ver si las
series se explican entre sí con un modelo multivariado.

Las tres series son:

- **Nieve**: superficie cubierta de nieve, derivada de imágenes Landsat. Es la más larga,
  471 meses entre enero de 1986 y marzo de 2025.
- **Agua**: superficie de cuerpos de agua, derivada de Sentinel-2. Desde 2016.
- **SAVI**: índice de vegetación ajustado por suelo, también de Sentinel-2. Desde 2016.

La cuenca está dividida en siete zonas (ZC1 a ZC4 y ZS1 a ZS3). Los datos se exportaron
desde Google Earth Engine a nivel de zona y acá se agregan a nivel de cuenca: se suman las
variables extensivas (las dos superficies) y se promedia la intensiva (SAVI). Como Sentinel-2
pasa cada cinco días y la cantidad de escenas útiles por mes varía entre dos y ocho, todo se
lleva a frecuencia mensual promediando dentro de cada mes.

Terminamos trabajando con dos ventanas distintas. La completa, de 471 meses, sólo tiene nieve
y se usa para el modelo univariado de esa serie. La común a las tres arranca en julio de 2016
y tiene 105 meses; es la que usa el VAR.

## Qué hay en cada notebook

**`TP1_01_carga_y_EDA.ipynb`** — Carga, control de calidad y análisis exploratorio. Acá se
resuelven las decisiones que condicionan todo lo que sigue: cómo agregar las zonas, qué hacer
con los meses faltantes, y con qué ventana trabajar. Incluye descomposición STL, perfil
estacional, análisis de estabilidad de la varianza, tendencia por Mann-Kendall y correlación
cruzada entre las series. Exporta las series mensuales listas para modelar.

Dos cosas que salieron de este notebook y que vale la pena mencionar: la serie de nieve tiene
una asimetría de +3,77, lo que obliga a trabajarla en logaritmo, y la de agua tiene cuatro
meses vacíos, dos de ellos dentro de la ventana común, que se imputan por interpolación
lineal.

**`TP1_02_estacionariedad.ipynb`** — Estacionariedad. Primero se estabiliza la varianza y
después la media, en ese orden. Correlogramas (FAS, FAC, FACP) antes y después de diferenciar,
y cuatro tests de raíz unitaria: ADF, KPSS, Phillips-Perron y DF-GLS, más el test de Zivot y
Andrews para detectar quiebres estructurales en fecha desconocida. También hay un control de
sobrediferenciación, porque diferenciar de más es un error tan caro como diferenciar de menos.

Los órdenes que quedaron: nieve en logaritmo y estacionaria en nivel (d=0, D=0); SAVI con una
diferencia regular y una estacional (d=1, D=1); agua con una diferencia regular (d=1, D=0).

**`TP1_03_modelos_sarima.ipynb`** — Modelos SARIMA. Búsqueda en grilla con selección por
criterios de información, evaluación sobre un conjunto de prueba, diagnóstico de residuos y
pronóstico. El test son los últimos 24 meses en la nieve y los últimos 12 en SAVI y agua.
Se comparan además contra Holt-Winters y contra un naive estacional, y contra la versión sin
componente estacional del mismo modelo, para verificar que la parte estacional aporta algo.

Los modelos elegidos fueron SARIMA(3,0,1)(1,0,1)[12] para la nieve, SARIMA(0,1,1)(0,1,1)[12]
para SAVI y SARIMA(1,1,1)(1,0,1)[12] para el agua.

**`TP1_04_var_causalidad.ipynb`** — Modelo VAR sobre las tres series en la ventana común,
test de causalidad de Granger, funciones impulso-respuesta y descomposición de la varianza del
error de pronóstico.

## Resultados

El hallazgo principal es la **causalidad de Granger de la nieve hacia el agua** (F = 4,51;
p = 0,035), sin evidencia en el sentido inverso ni hacia la vegetación. Tiene sentido físico
directo: la nieve acumulada en altura alimenta, al derretirse, la superficie de agua de la
cuenca. Es el resultado que le da unidad al trabajo, porque conecta dos series que hasta ese
punto se venían modelando por separado.

El segundo es el **quiebre estructural de mayo de 2019** en la serie de nieve, detectado por
el test de Zivot y Andrews. La serie cambia de régimen ahí, y el resto del análisis hay que
leerlo teniendo eso presente.

En cuanto al desempeño, el resultado más claro está en el agua: el VAR baja el MAE de 64,2 a
34,1, casi a la mitad. Es coherente con lo anterior —la nieve aporta información que el modelo
univariado no tiene—. En SAVI los dos empatan (0,037 contra 0,038), lo que también encaja: es
la serie donde no se encontró ninguna relación con las otras dos. Y en la nieve el VAR muestra
un MAE menor, pero ahí la comparación no es del todo pareja: al ser multivariado se evaluó
sobre 12 meses, mientras que el SARIMA se evaluó sobre 24, y pronosticar más cerca es más
fácil.

La contracara del VAR es que está limitado a la ventana común de 105 meses, mientras que el
SARIMA de la nieve aprovecha los 471.

Una advertencia sobre las métricas. El sMAPE queda muy alto en la nieve —arriba del 110 % en
todos los modelos— pero eso no significa que ninguno sirva. La serie tiene meses de verano con
superficie casi nula, y cuando el denominador del sMAPE se achica tanto, un error absoluto
chico produce un porcentaje enorme. Es una limitación de la métrica, no de los modelos.

## Estructura del repo

```
TP1/
├── DATASETS/                       datos crudos exportados de Google Earth Engine
├── OUTPUTS/                        salidas de cada notebook (CSV)
├── TP1_01_carga_y_EDA.ipynb
├── TP1_02_estacionariedad.ipynb
├── TP1_03_modelos_sarima.ipynb
├── TP1_04_var_causalidad.ipynb
└── TP1_Informe_Final_Grupo8.pdf    el informe entregado
```

Los notebooks se corren en orden. Cada uno lee las salidas del anterior desde `OUTPUTS/`, así
que no hace falta recalcular nada hacia atrás.

## Cómo correrlo

Están hechos para Google Colab con los datos en Drive, pero la celda de configuración busca la
carpeta `DATASETS` sola y también funciona en local. Si no la encuentra, hay que editar
`RUTA_DIRECTA` en la primera celda de código del notebook 01.

Las librerías son las habituales —pandas, numpy, matplotlib, scipy, statsmodels— más `arch`,
que aporta Phillips-Perron y DF-GLS. El notebook 02 lo instala si falta, y si la instalación
falla sigue adelante con ADF, KPSS y Zivot-Andrews.

## Limitaciones

Vale la pena dejarlas escritas, porque condicionan hasta dónde se puede leer el trabajo:

- Las tres series vienen de productos satelitales, no de mediciones en campo. La superficie
  de nieve depende del enmascaramiento de nubes, y meses muy nublados pueden quedar
  subestimados.
- La ventana común son nueve años. Es poco para separar un cambio de régimen de la
  variabilidad interanual normal, y por eso el quiebre de 2019 se reporta como hallazgo pero
  no se le atribuye una causa.
- El VAR se estimó con 92 observaciones de entrenamiento. Alcanza para un orden 1, que es el
  que eligieron los criterios de información, pero no da margen para mucho más.
- Causalidad de Granger no es causalidad en sentido físico. Lo que dice el test es que los
  valores pasados de la nieve ayudan a predecir el agua dentro de una especificación lineal,
  que es una afirmación bastante más modesta.

## Continuación

Este trabajo tiene una segunda parte, donde las mismas tres series se modelan con técnicas de
machine learning y deep learning (ensambles, LSTM, N-BEATS, Prophet) y se comparan contra los
resultados de acá.

---

Trabajo grupal — Grupo 8.
