# TP_DS_10 – Patrones de demanda mensual de agentes del MEM (CAMMESA)

Proyecto grupal del curso de Data Science (Fundación YPF – Jump Educación).
Análisis de la demanda eléctrica mensual de los agentes del Mercado Eléctrico Mayorista (MEM) de Argentina, con un modelo de clustering (K-Means), una segmentación alternativa en dos etapas y un modelo supervisado complementario.

**Grupo 10:** Martín Román y María Gracia Archilla

---

## 1. Pregunta de investigación

¿Existen grupos de agentes del MEM con patrones de demanda mensual similares (en nivel, estacionalidad y variabilidad), y en qué medida esos grupos coinciden con categorías reales del sistema eléctrico (tipo de agente, región)?

## 2. Datos

- **Fuente:** `Demanda_Mensual.xlsx` de CAMMESA, exportado a `DF - Hoja 1.csv`.
- **Tamaño:** 56.696 filas y 13 columnas, sin valores nulos. Contiene 557 agentes distintos y 47 meses (de enero de 2022 a noviembre de 2025). Cada fila corresponde a un agente, un mes y una categoría de demanda o tarifa, por lo que un mismo agente puede tener varias filas en el mismo mes. Hay 5 tipos de agente, 9 regiones y 3 categorías tarifarias.
- **Limpieza:** la demanda venía como texto con formato local (`1.537,96`) y con guiones en los meses sin demanda; se convirtió a número y los guiones se tomaron como 0 (se interpreta que ese mes el agente no tuvo demanda). La columna `MES` se convirtió a fecha.
- **Tabla intermedia:** una fila por agente y mes (suma de las filas del mismo agente y mes).

## 3. Metodología

1. **Ventana temporal común.** Año 2024, con los agentes que tienen demanda positiva en los 12 meses: **474 agentes**.
2. **Variables por agente** (una fila por agente):
   - `log_demanda_media`: nivel de demanda (logaritmo de la demanda media).
   - `cv` = desvío / media: variabilidad mes a mes, sin depender del tamaño del agente.
   - `ind_estacional` = (media de verano − media de invierno) / demanda media anual. Verano = diciembre, enero y febrero; invierno = junio, julio y agosto.
   - Se descartó `factor_carga` por su correlación con el CV.
3. **Revisión de la tabla de agentes:** valores atípicos y agentes con demanda intermitente (se conservan, con justificación), correlaciones, transformación logarítmica del CV y escalado con `RobustScaler`.
4. **K-Means** (`n_init=10`, `random_state=42`) con las variables `log_demanda_media`, `log_cv` e `ind_estacional`. Evaluación de k con el método del codo y comparación de perfiles (k = 3 y k = 4), interpretación de los clusters en unidades originales, visualización con PCA y prueba de estabilidad.
5. **Comparación** de los clusters con el tipo de agente y la región.
6. **Alternativa en dos etapas:** separar a los agentes con CV mayor a 1 y aplicar K-Means al grupo principal.
7. **Modelo supervisado:** clasificar el tipo de agente con las mismas variables (con el CV sin transformar), con regresión logística, árbol de decisión y Random Forest, y validación cruzada.

## 4. Decisiones y criterios

| Decisión | Criterio |
|---|---|
| Ventana 2024 | Es el año calendario con más agentes con registro en los 12 meses y demanda promedio positiva (474; en 2022 hay 454 y en 2023, 468). La ventana de diciembre de 2024 a noviembre de 2025 incluye 477 agentes, solo 3 más, y se prefirió un año calendario completo. Quedan fuera 83 de los 557 agentes |
| Se descartó `factor_carga` | Correlación de −0,88 con el CV: ambas miden en esencia qué tan irregular es el consumo, por lo que se conserva solo el CV |
| Se conservaron los outliers | No son errores de carga sino agentes reales con perfiles de consumo distintos; el 81 % de los Autogeneradores y el 33 % de los Distribuidores tiene alguno, frente al 11 % de los Grandes Usuarios. Eliminarlos implicaría perder casi toda la clase Autogenerador y distorsionar la comparación con las categorías reales |
| Se conservaron los agentes intermitentes | 18 agentes tienen entre 1 y 11 meses con demanda; 15 de los 31 Autogeneradores son intermitentes, frente a 3 de 366 Grandes Usuarios y ningún Distribuidor. Eliminarlos implicaría perder casi la mitad de la clase Autogenerador y sesgar el análisis |
| `log(cv)` | El CV original tiene una distribución muy asimétrica (asimetría 4,75) y esa cola larga haría que K-Means arme clusters para aislar a esos pocos agentes. El logaritmo reduce la asimetría (0,79); con la transformación el cluster mayoritario pasó de concentrar el 88 % de los agentes al 64 % |
| `RobustScaler` | Escalado de las tres variables del modelo |
| k = 4 | El quiebre más claro del codo está en k = 3, por lo que se compararon los perfiles de k = 3 y k = 4. Se eligió k = 4 por interpretabilidad: con k = 3, los agentes estables de tamaño moderado y los de gran tamaño quedan en un mismo cluster, mientras que con k = 4 se separa un grupo de 74 agentes grandes |
| Regla CV > 1 (alternativa) | Separa a los agentes cuyo desvío mensual supera a su demanda media (consumo muy irregular o intermitente) |
| No se usa la categoría tarifaria | No es una característica del agente: 77 agentes tienen más de un valor |
| Variable objetivo: `TIPO AGENTE` | Es una característica del agente: cada agente tiene un único valor. `CATEGORIA DEMANDA` y `TIPO DEMANDA` son propiedades de cada fila, por lo que un mismo agente puede aparecer con más de un valor |
| Autogenerador = AG + AR | AG y AR tienen muy pocos agentes (36 y 3), todos corresponden a empresas industriales con TIPO DEMANDA GUMA. Por su perfil similar y para evitar clases con casi ningún ejemplo, se unieron en una única clase |
| Se excluyó el tipo DM | Tiene un único agente: no se puede estratificar, dividir en entrenamiento y prueba ni validar de forma cruzada |
| F1 macro como métrica (parte supervisada) | Las clases están desbalanceadas, y el F1 macro asigna el mismo peso a cada clase independientemente de su tamaño |

## 5. Resultados

### 5.1 Demanda agregada del sistema

La demanda agregada del sistema presenta un patrón estacional que se repite todos los años, con dos picos (verano, alrededor de enero y febrero, e invierno, alrededor de junio y julio) y dos valles (abril y septiembre–octubre). Los picos de verano e invierno son de magnitud similar, por lo que el índice estacional del conjunto es cercano a 0: el índice identifica a los agentes que se apartan de ese patrón, con predominio de verano (valores positivos) o de invierno (valores negativos), y no mide la estacionalidad en general.

### 5.2 Clusters (K-Means, k = 4, inercia 967,7)

| Cluster | Perfil | Agentes | % de agentes | % de la demanda |
|---|---|---|---|---|
| 0 | Demanda moderada (mediana 1.626), muy estable (CV 0,15), sin estacionalidad | 304 | 64,1 % | 6,2 % |
| 1 | Agentes de gran tamaño (mediana 30.501), muy estables (CV 0,12), sin estacionalidad | 74 | 15,6 % | 92,5 % |
| 2 | Demanda baja (mediana 681), variabilidad alta (CV 0,69), más demanda en verano (índice estacional +0,57) | 79 | 16,7 % | 1,1 % |
| 3 | Demanda baja (mediana 753), variabilidad muy alta (CV 1,10), más demanda en invierno (índice estacional −1,39) | 17 | 3,6 % | 0,2 % |

El paso de k = 3 a k = 4 separa a los agentes estables según su tamaño: el cluster 0 de k = 3 (370 agentes) se desdobla en gran parte en un grupo de demanda media-baja (304 agentes) y otro de demanda muy alta (74 agentes). Los clusters de estacionalidad positiva (86 → 79 agentes) y negativa con CV alto (18 → 17 agentes) casi no cambian.

**PCA (solo para visualizar).** Con dos componentes retiene el 86,2 % de la información de las tres variables (54,3 % el primer componente y 31,9 % el segundo). El primer componente corresponde casi enteramente a la estacionalidad (peso 1,00) y separa a los agentes con más demanda en verano de los de invierno. El segundo contrasta demanda media y variabilidad (+0,79 en `log_demanda_media` y −0,62 en `log_cv`) y distingue a los agentes grandes y estables de los pequeños y variables. En el plano de los dos componentes, el cluster 3 (invierno) queda a la izquierda y el cluster 2 (verano) a la derecha; el cluster 1 ocupa la parte superior; el cluster 0 forma una masa central compacta que se solapa parcialmente con el cluster 1.

### 5.3 Coincidencia con las categorías reales

**Tipo de agente** (n = 473; DM excluido):

| Cluster | Autogenerador | Distribuidor | Gran Usuario |
|---|---|---|---|
| 0 | 3 | 20 | 280 |
| 1 | 2 | 53 | 19 |
| 2 | 18 | 3 | 58 |
| 3 | 8 | 0 | 9 |

- El cluster 1, de agentes grandes y estables, está compuesto en un 71,6 % por Distribuidores (53 de 74), frente al 16,1 % que representan en la muestra completa, y reúne al 70 % de todos los Distribuidores.
- El cluster 0 está formado por Grandes Usuarios en un 92,4 %.
- Los Autogeneradores se concentran en los clusters de comportamiento variable y estacional (26 de 31 en los clusters 2 y 3), en particular el cluster 3, donde son el 47,1 % de 17 agentes, un grupo pequeño que debe interpretarse con cautela.
- La coincidencia es parcial: el 30 % de los Distribuidores queda fuera del cluster 1, y el cluster 2 sigue siendo mayoritariamente Gran Usuario.
- Esto indica que el tipo de agente se asocia a patrones de demanda característicos, pero no los determina.

**Región** (9 regiones). La coincidencia es débil:

- Cluster 0: predomina el Gran Buenos Aires (42,2 %).
- Cluster 1: tiene más agentes de la provincia de Buenos Aires (37,8 %) y menos del Gran Buenos Aires (8,1 %).
- Cluster 2: el Gran Buenos Aires sigue siendo la región más frecuente (27,8 %) y el Noroeste está algo por encima (15,2 %).
- Cluster 3: son solo 17 agentes; la región más frecuente es la provincia de Buenos Aires (29,4 %), seguida de la Patagónica (23,5 %).

En conjunto, la región se asocia con los patrones de demanda menos que el tipo de agente.

### 5.4 Alternativa: segmentación en dos etapas

- **Etapa 1 (regla CV > 1):** se separan 21 agentes (4,4 % de los 474), que representan apenas el 0,06 % de la demanda total. 17 de ellos son agentes intermitentes (de los 18 que hay en total): son 17 Autogeneradores y 4 Grandes Usuarios, y ningún Distribuidor.
- **Etapa 2:** K-Means con k = 4 sobre el grupo principal (453 agentes), con las mismas variables escaladas de nuevo solo con este grupo. En este grupo, el método del codo muestra el quiebre más claro en k = 4.

| Cluster | Perfil | Agentes | % de agentes del grupo principal | % de la demanda |
|---|---|---|---|---|
| 0 | Demanda moderada (mediana 2.030), muy estable (CV 0,11), sin estacionalidad (índice +0,04) | 227 | 50,1 % | 5,5 % |
| 1 | Demanda moderada-baja (mediana 1.284), variabilidad media (CV 0,29), más demanda en invierno (índice −0,32) | 83 | 18,3 % | 1,8 % |
| 2 | Agentes de gran tamaño (mediana 52.389), muy estables (CV 0,13), sin estacionalidad (índice +0,05) | 62 | 13,7 % | 91,5 % |
| 3 | Demanda baja (mediana 967), variabilidad media (CV 0,36), más demanda en verano (índice +0,47) | 81 | 17,9 % | 1,3 % |

| Cluster | Autogenerador | Distribuidor | Gran Usuario |
|---|---|---|---|
| 0 | 1 | 23 | 202 |
| 1 | 3 | 2 | 78 |
| 2 | 2 | 48 | 12 |
| 3 | 8 | 3 | 70 |

Los Distribuidores se concentran en el cluster 2, de agentes de gran tamaño y muy estables: es un 77,4 % Distribuidores y reúne a 48 de los 76 Distribuidores. Los Grandes Usuarios, en cambio, se reparten en los otros tres clusters (89 a 94 % de Grandes Usuarios), que se diferencian por su variabilidad, su tamaño y su estacionalidad: estables y sin estacionalidad (cluster 0), con más demanda en invierno (cluster 1) y con más demanda en verano (cluster 3). En el modelo con todos los agentes, 280 de los 366 Grandes Usuarios caían en un único cluster de agentes estables y sin estacionalidad; la segunda etapa los reparte entre tres perfiles distintos.

La separación entre Distribuidores y Grandes Usuarios es clara pero no total: 12 Grandes Usuarios (19,4 % del cluster 2) tienen un perfil de gran tamaño y estable como los Distribuidores, y 23 Distribuidores (30 %) están en el cluster 0. Quedan solo 14 Autogeneradores, por lo que no se interpretan por separado.

### 5.5 Estabilidad del algoritmo

Para evaluar la estabilidad de K-Means con k = 4 se entrenó el modelo con 10 semillas distintas y un solo intento de inicialización (`n_init=1`). La solución final (inercia 967,7), obtenida repitiendo 10 inicializaciones y conservando la mejor, tiene tamaños de cluster [304, 79, 74, 17]. La semilla 2 (inercia 967,7) tuvo la misma inercia y tamaños casi idénticos. En 4 de las 10 semillas (0, 3, 5 y 7) apareció una solución alternativa que aísla a 4 agentes extremos, con inercia mayor (972,8 a 989,2). En las restantes, los tamaños de los clusters variaron según la semilla. La estructura general es estable, pero los límites exactos entre clusters dependen de la inicialización, por lo que se usó `n_init=10`.

### 5.6 Modelo supervisado (tipo de agente)

Objetivo: clasificar el tipo de agente (Gran Usuario, Distribuidor y Autogenerador; 473 agentes) con `log_demanda_media`, `cv` (sin transformar) e `ind_estacional`. División en entrenamiento (378 agentes) y prueba (95 agentes), estratificada, y validación cruzada de 5 particiones.

| Modelo | F1 macro (validación cruzada) | Desvío |
|---|---|---|
| Base (Dummy) | 0,291 | 0,001 |
| Regresión logística | 0,661 | 0,078 |
| Árbol de decisión | 0,665 | 0,109 |
| Random Forest | 0,745 | 0,072 |

Random Forest obtuvo el mayor F1 macro promedio, y los tres modelos superan ampliamente al modelo base. Sin embargo, los desvíos entre particiones (0,07 a 0,11) son del orden de las diferencias entre modelos.

**Ajuste de hiperparámetros** (`GridSearchCV`): el mejor modelo (300 árboles, ninguna profundidad máxima y mínimo de 1 agente por hoja) obtiene un F1 macro de 0,753 en validación cruzada (desvío 0,086), frente a 0,745 del Random Forest inicial; la diferencia es del orden del desvío entre las particiones.

**Evaluación final en el conjunto de prueba (95 agentes):** el Random Forest final obtiene un F1 macro de 0,745.

| Clase | Precisión | Sensibilidad | Agentes en prueba |
|---|---|---|---|
| Gran Usuario | 0,92 | 0,82 | 74 |
| Distribuidor | 0,50 | 0,73 | 15 |
| Autogenerador | 0,71 | 0,83 | 6 |

La precisión de Distribuidor es la más baja: de 22 agentes predichos como Distribuidor, solo 11 lo son, porque 11 Grandes Usuarios se clasificaron como Distribuidor. Autogenerador tiene solo 6 agentes en el conjunto de prueba, por lo que sus métricas son muy poco precisas.

**Sobreajuste:** el Random Forest final obtuvo un F1 macro de 0,963 en el conjunto de entrenamiento (calculado sobre los mismos agentes con los que se entrenó), 0,753 en validación cruzada y 0,745 en el conjunto de prueba. La diferencia entre entrenamiento y validación indica sobreajuste. El desempeño en prueba es similar al de validación cruzada, lo que sugiere que el modelo se comporta de forma consistente con agentes no vistos durante el entrenamiento; sin embargo, el conjunto de prueba es pequeño, por lo que esa estimación es poco precisa.

## 6. Conclusiones

- **Pregunta de investigación.** Existen grupos de agentes del MEM con patrones de demanda mensual similares. Con K-Means (k = 4) sobre el nivel de demanda, la variabilidad (log del CV) y la estacionalidad, se identificaron cuatro grupos: agentes de demanda moderada y estable (64,1 % de los agentes, 6,2 % de la demanda); agentes grandes y estables (15,6 % de los agentes, pero 92,5 % de la demanda); agentes pequeños y variables con más demanda en verano (16,7 %); y agentes pequeños y muy variables con más demanda en invierno (3,6 %).
- **Coincidencia con categorías reales.** La coincidencia es clara con el tipo de agente y débil con la región. El grupo de agentes grandes y estables está compuesto en un 71,6 % por Distribuidores y reúne al 70 % de todos los Distribuidores; los grupos restantes son mayoritariamente Grandes Usuarios, y los Autogeneradores se concentran en los grupos variables y estacionales. La coincidencia no es total: el tipo de agente se asocia a patrones de demanda característicos, pero no los determina.
- **Transformación del CV y alternativa en dos etapas.** La transformación logarítmica del CV evitó que K-Means dedicara clusters a aislar casos extremos: el cluster mayoritario pasó de concentrar el 88 % de los agentes a concentrar el 64 %. Como alternativa, se separaron los 21 agentes con CV mayor a 1 (0,06 % de la demanda) y se aplicó K-Means al grupo principal: los Distribuidores quedaron concentrados en un cluster (77,4 %), mientras que los Grandes Usuarios se repartieron en tres perfiles que se diferencian por variabilidad, tamaño y estacionalidad.
- **Modelo supervisado.** Un Random Forest que clasifica el tipo de agente con las mismas variables obtuvo un F1 macro de 0,753 en validación cruzada y 0,745 en prueba, frente a 0,291 del modelo base. La clase Distribuidor tiene una precisión baja (0,50) y los resultados de Autogenerador son poco precisos por su escaso número de agentes.

## 7. Limitaciones

- El análisis usa un único año (2024) y solo agentes con registro en los 12 meses (quedan fuera 83 de los 557 agentes).
- Para los agentes intermitentes, las métricas (en particular el CV y el índice estacional) se calculan sobre pocos meses.
- El índice estacional conserva valores extremos aun después del escalado.
- La elección de k = 4 combina el método del codo con la interpretabilidad. La estabilidad con distintas inicializaciones muestra que la estructura general se mantiene, aunque los límites exactos entre clusters varían.
- El cluster 3 tiene solo 17 agentes y debe interpretarse con cautela.
- La categoría tarifaria no se usó porque no es una característica del agente.
- La clase Autogenerador (31 agentes, resultado de unir AG y AR) y el conjunto de prueba (95 agentes) son pequeños, por lo que las conclusiones sobre ellas deben considerarse con precaución.

## 8. Estructura del repositorio

```
├── DATA/
│   ├── DF - Hoja 1.csv             # datos de CAMMESA
│   └── TP_DS_YPF_3ER.ipynb         # análisis completo
├── requirements.txt                # librerías y versiones
└── README.md
```

## 9. Cómo reproducir

1. Clonar el repositorio y crear un entorno virtual.
2. Instalar las dependencias: `pip install -r requirements.txt`.
3. Abrir `DATA/TP_DS_YPF_3ER.ipynb` con el kernel del entorno y ejecutar todas las celdas en orden. El notebook lee `DF - Hoja 1.csv` desde su misma carpeta y la semilla aleatoria está fija en 42.