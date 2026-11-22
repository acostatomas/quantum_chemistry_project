# Simulación de la disociación de H₄ con VQE

Proyecto desarrollado por el equipo **Sycamore** para **Quantum Jam 2026**.

Estudiamos la energía de una cadena de cuatro átomos de hidrógeno mediante el algoritmo **Variational Quantum Eigensolver (VQE)** y comparamos los resultados con **CCSD** y **Full Configuration Interaction (FCI)**.

En la corrida final, VQE superó a CCSD en los **nueve puntos de la zona estirada**, incluyendo una barra de incertidumbre estadística en el criterio de comparación. La estimación central alcanzó precisión química en **tres de esos puntos**.

## Equipo

- Tomás Acosta
- Felipe Bridge
- Ariel Duarte

## Objetivo

La ruptura de enlaces puede requerir una superposición de varias configuraciones electrónicas importantes. Esto dificulta la descripción del sistema mediante métodos aproximados basados en una única referencia.

Usamos H₄ como un sistema pequeño para estudiar este problema y evaluar cómo influyen el diseño del circuito, la optimización y el presupuesto de mediciones en la precisión de VQE.

El desafío consiste en:

1. Calcular la curva de energía para distintas separaciones entre átomos.
2. Estimar la distancia de equilibrio y la energía de disociación.
3. Superar a CCSD para distancias desde 1,80 Å.
4. Evaluar la confiabilidad del resultado mediante controles físicos y estadísticos.

## Sistema estudiado

| Propiedad | Configuración |
|---|---|
| Molécula | H₄ neutro |
| Geometría principal | Cadena recta con separación uniforme |
| Posiciones sobre el eje z | 0, d, 2d y 3d |
| Distancias | 17 puntos entre 0,60 y 3,00 Å |
| Base | STO-3G |
| Electrones | 4, distribuidos en 2 alfa y 2 beta |
| Orbitales espaciales | 4 |
| Mapeo | Jordan–Wigner |
| Qubits | 8 |
| Entorno de ejecución | Simulación clásica |

FCI proporciona la referencia exacta dentro de la base y del modelo molecular elegidos.

## Metodología

### Preparación del problema

Utilizamos PySCF para calcular Hartree–Fock restringido, las integrales moleculares y las referencias CCSD y FCI.

Para cada geometría construimos un Hamiltoniano de qubits como suma de operadores de Pauli, incluyendo la repulsión nuclear. Verificamos que reproduzca la energía de Hartree–Fock y la energía FCI en el sector de dos electrones alfa y dos beta.

### Ansatz y optimización

Construimos un ansatz químico tipo UCCSD que parte de Hartree–Fock y aplica excitaciones simples y dobles. Cada capa tiene 26 parámetros y conserva la cantidad de electrones de cada spin.

| Variante | Parámetros | Compuertas CZ tras la transpilación |
|---|---:|---:|
| Una capa | 26 | 292 |
| Dos capas | 52 | 582 |

Optimizamos los ángulos con **L-BFGS-B**, usando energías y gradientes calculados mediante matrices dispersas.

Para cada geometría probamos el estado inicial de Hartree–Fock, los parámetros de la distancia anterior y tres inicializaciones aleatorias. Verificamos la implementación rápida comparando sus energías con las del circuito de Qiskit.

La curva final medida corresponde al ansatz de **una capa**. La variante de dos capas se estudió también durante la optimización sin ruido.

### Medición con shots

Las energías reportadas se obtienen con `SamplerV2` de Qiskit Aer sobre circuitos ideales.

El procedimiento:

1. Agrupa términos de Pauli compatibles con una misma base de medición.
2. Realiza un piloto de 4.000 shots por grupo.
3. Distribuye los shots finales en proporción a la desviación estándar estimada de cada grupo.
4. Calcula la energía y su incertidumbre con una muestra final independiente.

El presupuesto principal es de **2.000.000 de shots por geometría**, incluyendo el piloto. Para los 46 grupos de la medición, el piloto consume 184.000 shots y quedan 1.816.000 para la estimación final.

La incertidumbre se calcula mediante:

$$
\sigma_E^2 = \sum_g \frac{s_g^2}{N_g}
$$

Aquí, cada varianza corresponde al aporte conjunto del grupo, incorporando las covarianzas entre términos medidos juntos.

## Resultados

Para superar a CCSD aplicamos el criterio del desafío:

$$
|E_{\mathrm{VQE}}-E_{\mathrm{FCI}}|+\sigma_E
<
|E_{\mathrm{CCSD}}-E_{\mathrm{FCI}}|
$$

| d (Å) | Error VQE + σ (mHa) | Error absoluto CCSD (mHa) | Supera a CCSD | Precisión química |
|---:|---:|---:|:---:|:---:|
| 1,80 | 3,551 | 9,107 | Sí | No |
| 1,95 | 4,635 | 15,929 | Sí | No |
| 2,10 | 4,871 | 22,749 | Sí | No |
| 2,25 | 2,347 | 27,857 | Sí | Sí |
| 2,40 | 4,691 | 30,560 | Sí | No |
| 2,55 | 2,183 | 30,943 | Sí | Sí |
| 2,70 | 5,124 | 29,445 | Sí | No |
| 2,85 | 2,779 | 26,619 | Sí | Sí |
| 3,00 | 3,257 | 23,013 | Sí | No |

La precisión química se evalúa sobre el **error central**, con un umbral de 1,6 mHa. Por eso puede alcanzarse aunque la columna que suma σ supere ese umbral.

CCSD subestimó la energía FCI en los nueve puntos estirados. Su energía carece de garantía variacional.

### Equilibrio y disociación

El mínimo de la grilla medida se encuentra en **d = 0,90 Å**. Usando directamente las energías medidas:

$$
D_e = E(3{,}00\,\text{Å})-E(0{,}90\,\text{Å})
\approx 0{,}311684\,\text{Ha}
$$

También exploramos un ajuste parabólico y propagamos la incertidumbre estadística mediante 5.000 realizaciones Monte Carlo.

La distancia estimada mostró sensibilidad a la ventana del ajuste:

| Puntos utilizados | Distancia del vértice (Å) |
|---:|---:|
| 3 | 0,9051 |
| 5 | 0,9580 |
| 7 | 1,0265 |

Esta sensibilidad supera la incertidumbre estadística estimada por Monte Carlo. Refinar la distancia de equilibrio requiere más geometrías alrededor del mínimo.

## Controles de confiabilidad

Para d = 2,40 Å, con una capa:

| Diagnóstico | Resultado |
|---|---:|
| Mejora energética sin ruido frente a Hartree–Fock | 436,424 mHa |
| Número esperado de electrones | 4,000 |
| Peso en el sector de 2 alfa y 2 beta | 1,000 |
| Varianza energética de Hartree–Fock | 0,147954 Ha² |
| Varianza energética de VQE | 0,000381 Ha² |
| σ con 2 millones de shots totales | 1,405 mHa |
| σ con 8 millones de shots totales | 0,676 mHa |

Las dos estimaciones medidas difirieron en 0,32 incertidumbres combinadas.

Los diagnósticos del estado se calcularon en simulación exacta. Una varianza energética pequeña indica poca dispersión en energía y debe interpretarse junto con los demás controles para evaluar la aproximación al estado fundamental.

## Extensión a H₄ cuadrado

Exploramos cuatro hidrógenos en los vértices de un cuadrado, reutilizando la construcción del Hamiltoniano y la optimización.

Con dos capas obtuvimos energías muy próximas a FCI en simulación sin ruido. Esta extensión presenta resultados de optimización y sirve para estudiar el efecto de la geometría sobre el desempeño de los métodos.

## Herramientas

- **Python y NumPy** para el procesamiento numérico.
- **PySCF** para Hartree–Fock, integrales moleculares, CCSD y FCI.
- **Qiskit** para operadores, circuitos y transpilación.
- **Qiskit Aer** para el muestreo con shots.
- **SciPy** para la optimización.
- **Pandas y Matplotlib** para tablas y gráficos.

## Ejecución

Abrir `Copia_de_QNTJAM_Trabajo.ipynb` en Google Colab o Jupyter e instalar las dependencias:

```python
%pip install "qiskit[visualization]" qiskit-aer pyscf scipy numpy pandas matplotlib
```

Ejecutar las celdas en orden desde un kernel limpio. El barrido completo incluye varias optimizaciones y millones de shots por geometría.

El notebook conserva distintas versiones experimentales de algunas funciones. Al volver a ejecutar una celda anterior después de redefinir una función, puede cambiar la implementación utilizada.

Además:

- La optimización de una capa sobrescribe los campos generales de ángulos y energía de la corrida previa de dos capas.
- Las mediciones existentes se reutilizan. Si se cambian los ángulos o las capas, deben recalcularse las mediciones correspondientes.
- Algunos textos del notebook conservan cifras de corridas anteriores. La tabla de este README corresponde a las salidas de la corrida final de una capa del notebook adjunto.

## Limitaciones y mejoras

- La simulación ideal incluye incertidumbre de muestreo. Evaluar hardware requiere incorporar errores de compuertas y lectura.
- El costo de 292 CZ por circuito de una capa puede resultar exigente en dispositivos ruidosos.
- La optimización mediante matrices y vectores de estado tiene un costo que crece con el tamaño del sistema.
- La precisión química se alcanzó en tres de los nueve puntos estirados.
- El ajuste de equilibrio requiere una grilla más densa cerca del mínimo.
- La conservación de resultados por configuración y de todos los arranques puede mejorarse para facilitar la reproducibilidad.

Como próximos pasos, proponemos seleccionar excitaciones de mayor impacto, reoptimizar circuitos más cortos, medir la variante de dos capas y evaluar el comportamiento con modelos de ruido.
