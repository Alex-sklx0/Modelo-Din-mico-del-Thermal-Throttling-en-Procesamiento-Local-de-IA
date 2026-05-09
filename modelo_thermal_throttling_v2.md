# Modelo Dinámico del Thermal Throttling en Procesamiento Local de IA

# Equipo

# Responsabilidades:

# Resumen


# 1. Introducción

## 1.1 Presentación del sistema
El sistema escogido es la unidad de procesamiento de una computadora (CPU/NPU) operando bajo cargas de trabajo generadas por la inferencia de Modelos de Lenguaje Grande (LLM). Este sistema es un entorno dinámico donde la potencia eléctrica, la temperatura de los materiales como el silicio y la velocidad de procesamiento de la IA (medida en tokens por segundo) se  acoplan. A diferencia de otros sistemas mecánicos, aquí las decisiones se toman en microsegundos y de forma no lineal ya que el sistema presenta un comportamiento de 2 regímenes: una fase de aceleración suave y una fase de desaceleracion extrema. Estas fases se dan por la forma en la que esta construido el hardware, que tiene un firmware protector cuando alcanza sus límites térmicos, disimuyendo o cortando abruptamente la fuente de energia hacia el procesador para evitar daños fisicos. 

## 1.2 Relevancia

Con los estándares actuales de hardware para IA local (requisito industrial de $>40$ TOPS), un equipo ejecutando un Modelo de Lenguaje Grande (LLM) está siempre en tensión física gracias a 3 fuerzas:El sistema operativo busca maximizar la productividad asignando la mayor potencia posible para generar tokens con baja latencia.El procesador, por efecto Joule, incrementa su temperatura de forma proporcional a esa carga de trabajo.El firmware interviene como un mecanismo de seguridad recortando la energía de forma drástica para prevenir daños irreversibles en la arquitectura del chip.

Definir un equilibrio entre las 3 fuerzas asegura la viabilidad operativa de cualquier equipo dedicado a la inteligencia artificial.

## 1.3 Fenómeno de estudio: 

El Thermal Throttling es el fenómeno central. En términos de sistemas este se define como una penalización que impone el hardware para desacelerar el flujo de datos y reducir la generación de calor. En el contexto de la IA, este fenómeno genera oscilaciones severas. Cuando el equipo entra en throttling, el rendimiento se desploma por debajo de los umbrales mínimos de usabilidad (frecuentemente por debajo de 5 tokens/s), para luego intentar recuperarse una vez la temperatura desciende, creando un ciclo de inestabilidad que compromete la integridad del procesamiento. Modelar este fenómeno permite predecir el "punto de fatiga" del hardware y diseñar intervenciones que optimicen la longevidad del equipo sin sacrificar su capacidad de cómputo.


### 1.3.1 Por Qué No Existe Solución Analítica

El sistema **no admite solución analítica exacta** por dos razones:

El firmware del hardware actúa como un interruptor condicional que solo se activa cuando la temperatura sobrepasa un límite crítico, lo que divide el espacio de estados en dos comportamientos distintos: un régimen libre, donde el sistema opera según las demandas de rendimiento, y un régimen de protección, donde se introducen fuerzas de frenado para preservar el silicio.

El término de throttling se peude representar como un función de $\max(0, T - T_{crit})$ gracias a que el firmware no realiza ninguna acción sino hasta que la temperatura actual sobrepasa un umbral de temperatura critico.

El obstáculo matemático fundamental es que el instante exacto en el que ocurre este cambio, que lo llamaremos $t^*$ tal que $T(t^*) = T_{crit}$, es una variable desconocida que depende exclusivamente de la trayectoria previa del sistema. Al no poder predecir con exactitud cuándo o cuántas veces el sistema alternará entre estos estados, es imposible formular una expresión analítica que unifique todo. 

Además, el calor generado, la energía consumida y el rendimiento computacional están muy acoplados y la alteración de una afecta instantáneamente a las demás. Esto impide desacoplar las ecuaciones para encontrar una solucion generla, reforzando la necesidad de emplear aproximaciones numéricas como única metodología válida para analizar el comportamiento global del equipo.


## 1.4 Objetivos

El objetivo principal es modelar matemáticamente y simular mediante métodos numéricos la dinámica térmica y de rendimiento de un procesador bajo carga de IA, identificando las condiciones bajo las cuales el sistema alcanza un estado de equilibrio estable. 

Como objetivos específicos, se busca implementar un algoritmo de Runge-Kutta de cuarto orden (RK4) programado desde cero para resolver el sistema de ecuaciones diferenciales acopladas. 

Asimismo, se pretende realizar un análisis comparativo de la convergencia y el error variando el tamaño de paso de integración y validando los resultados contra un solucionador de referencia de alto orden. 

Finalmente se animan las trayectorias resultantes para interpretar visualmente la sensibilidad del hardware ante diferentes parámetros de refrigeración y configuración de software.

## 1.5 Estructura del informe

1. La metodología donde se desglosa el modelo matemático de tres ecuaciones diferenciales y se justifican las constantes físicas basadas en la arquitectura del sistema. 
2. Resultados y discusión presentan las simulaciones de los cuatro escenarios de intervención propuestos, acompañadas del análisis de error y estabilidad del método numérico empleado.
3. Síntesis de los hallazgos más relevantes, evaluando la efectividad de las mejoras de hardware y software para mitigar el impacto del throttling, y reflexiona sobre los aprendizajes técnicos obtenidos durante el proceso de implementación y validación.

---

# 2. Metodologia

## 2.1 Modelo matematico: Sistema de Ecuaciones Diferenciales

El fenómeno se modela como un sistema de tres ecuaciones diferenciales no lineales de primer orden. Las tres variables de estado son:

- $T(t)$: temperatura del procesador en el instante $t$ medida en °C
- $P(t)$: potencia eléctrica asignada al procesador medida en W
- $I(t)$: velocidad de inferencia, es decir, tokens generados por segundo medida en tokens/s

Las tres están acopladas: la potencia determina cuánto se calienta el procesador y cuán rápido procesa; la temperatura activa el throttling que recorta la potencia; la potencia recortada reduce la inferencia, lo que impulsa al sistema operativo a pedir más potencia. 

Aquí el texto unificado, incorporando las justificaciones de origen de cada ecuación sin duplicar contenido:

---

### Ecuación 1: Dinámica Térmica

$$\frac{dT}{dt} = k_1 P(t) - k_2 \bigl(T(t) - T_{amb}\bigr)$$

Esta ecuación describe la **evolución de la temperatura del procesador** en el tiempo. Es una aplicación directa del Modelo de Capacitancia Térmica Concentrada (*Lumped Capacitance Model*), donde el procesador se trata como un nodo único con una masa térmica $C_{th}$ y una resistencia térmica $R_{th}$ hacia el ambiente. El balance de energía canónico de este modelo es $C_{th} \frac{dT}{dt} = \dot{E}_{in} - \dot{E}_{out}$, donde la energía entrante es la potencia eléctrica disipada como calor (Efecto Joule) y la energía saliente sigue la Ley de Enfriamiento de Newton. Dividiendo entre $C_{th}$ se obtiene la forma exacta de la ecuación, con $k_1 \equiv \frac{1}{C_{th}}$ y $k_2 \equiv \frac{1}{R_{th}C_{th}}$.

El término $k_1 P(t)$ es la **tasa de calentamiento**: la potencia eléctrica consumida se disipa parcialmente como calor. La constante $k_1$ (°C/J) cuantifica cuántos grados de temperatura produce cada julio de energía entregada; depende de la arquitectura del chip a través de $\kappa_0$, que agrupa la ineficiencia térmica de la litografía del silicio.

El término $-k_2(T - T_{amb})$ representa la **disipación térmica** hacia el ambiente. La constante $k_2$ (s⁻¹) cuantifica la eficiencia del sistema de disipación: cuando $T = T_{amb}$ el procesador no pierde calor neto; cuando $T \gg T_{amb}$ la disipación es máxima. Depende físicamente del flujo de aire $\Phi_{aire}$ y la conductancia del disipador $H_d$.

*Referencia: Incropera, F. P., & DeWitt, D. P. — Fundamentos de transferencia de calor y masa, cap. Conducción transitoria.*

---

### Ecuación 2: Dinámica de Inferencia

$$\frac{dI}{dt} = \gamma P(t) - \delta I(t)$$

Esta ecuación describe la **evolución de la velocidad de inferencia** del modelo de lenguaje. Su forma proviene de los modelos de fluidos en teoría de colas (*Fluid Queueing Model*), donde la tasa de cambio de la variable de rendimiento depende de una tasa de inyección de trabajo y una tasa de decaimiento proporcional al nivel actual de saturación. La ecuación general de estos modelos es $\frac{dx}{dt} = \lambda(t) - \mu x(t)$, donde $\lambda(t)$ es la tasa de llegada de trabajo y $\mu x(t)$ la fricción proporcional a la carga actual. La variable de estado $I(t)$ actúa como la variable de flujo $x(t)$.

El término $\gamma P(t)$ representa la **aceleración computacional**: a mayor potencia entregada, mayor frecuencia de operación y mayor throughput de cálculo matricial, que es la operación central en los LLMs. La constante $\gamma$ mide la eficiencia con que la energía se convierte en cómputo útil e incorpora las Instrucciones Por Ciclo (IPC) de la arquitectura y el número de hilos lógicos disponibles.

El término $-\delta I(t)$ representa la **fricción computacional**: ningún sistema acelera indefinidamente. Al aumentar la velocidad de inferencia, se satura el ancho de banda de memoria RAM y los buses de la placa base. Esta saturación introduce un decaimiento natural proporcional a la velocidad actual $I(t)$. La inferencia de LLMs es un proceso severamente limitado por el ancho de banda de memoria (*memory bound*), por lo que $\delta$ es inversamente proporcional a la velocidad del enlace más lento en la ruta de datos, ya sea la RAM (MT/s) o el bus PCIe (GT/s).

*Referencia: Kleinrock, L. — Queueing Systems, Vol. II: Computer Applications, cap. Fluid Approximations.*

---

### Ecuación 3: Control de Potencia con Throttling No Lineal

$$\frac{dP}{dt} = \alpha\bigl(I_{obj} - I(t)\bigr) - \beta \max\bigl(0,\, T(t) - T_{crit}\bigr)^3$$

Esta ecuación es la **lógica de control de potencia** del sistema operativo y el firmware, y contiene el término no lineal que hace al sistema matemáticamente irresoluble de forma analítica.

El término $\alpha(I_{obj} - I(t))$ representa la **demanda del sistema operativo**. En la teoría de control automático, este término corresponde exactamente a la acción de un controlador proporcional (componente P de un PID), donde la ganancia $\alpha$ actúa sobre el error $e(t) = I_{obj} - I(t)$ para ajustar la variable manipulada $P$. Cuando $I(t)$ está por debajo del objetivo, el planificador de energía incrementa la potencia proporcionalmente al déficit. Un $\alpha$ alto corresponde a un perfil de "Máximo rendimiento" que sube la potencia rápidamente; un $\alpha$ bajo, a un perfil conservador como "Ahorro de batería".

El término $-\beta\max(0, T - T_{crit})^3$ modela el **throttling del firmware**. En la teoría de control no lineal, este término corresponde a una función de barrera asimétrica de un solo lado (*one-sided penalty function*): mientras $T \leq T_{crit}$ el término es exactamente cero y el firmware no interviene; en el instante en que $T$ supera $T_{crit}$, el término se activa y crece con la tercera potencia del exceso $\Delta T = T - T_{crit}$, imponiendo una penalización cúbica sobre la potencia asignada. La elección del exponente cúbico se justifica por tres razones técnicas:

**Comportamiento físico del hardware.** La documentación técnica de Intel establece que las acciones ante una condición térmica crítica escalan según la severidad del evento, desde reducir el ancho de banda y la frecuencia hasta apagar subsistemas completos. En la práctica, el procesador no recorta su rendimiento proporcionalmente sino que transiciona de golpe hacia P-States inferiores a medida que el calor persiste. Brooks y Martonosi, en *Dynamic Thermal Management for High-Performance Microprocessors*, documentan que mecanismos como el *clock gating* global o la reducción dinámica de voltaje (DVFS) imponen caídas de rendimiento asimétricas cuya severidad aumenta polinómicamente frente a la magnitud de la violación térmica. La curva cúbica abstrae esa cascada: es tolerante a micro-fluctuaciones en la frontera del umbral pero impone un colapso masivo ante excesos sostenidos.

**Continuidad geométrica en el cruce de régimen.** La función $\max(0, \Delta T)^p$ tiene en $\Delta T = 0$ una pendiente y una curvatura ambas iguales a cero si y solo si $p \geq 3$. Con $p = 1$ la primera derivada es discontinua; con $p = 2$ la segunda derivada es discontinua. Con $p = 3$ tanto la pendiente como la curvatura son continuas y nulas exactamente en $T_{crit}$. Nocedal y Wright, en *Numerical Optimization*, establecen que exponentes superiores al cuadrático se justifican cuando el método de resolución numérica requiere derivadas continuas de orden mayor para no degradar su convergencia local. RK4 construye su aproximación interpolando pendientes en puntos intermedios del intervalo $[t_n, t_{n+1}]$; si la curvatura cambia abruptamente en $T_{crit}$, la extrapolación interna introduce un error de truncamiento local que contamina la solución en esa vecindad. El exponente 3 elimina esa discontinuidad y permite que el algoritmo transite el umbral sin perturbaciones geométricas que amplíen el error.

**Coherencia con la teoría de penalización exterior.** Rao, en *Engineering Optimization: Theory and Practice*, explicita que aunque $p = 2$ es el valor habitual por simplicidad analítica, el uso de $p = 3$ o $p = 4$ es completamente legítimo y necesario cuando la estabilidad del método numérico exige evitar discontinuidades en la curvatura justo en el límite de la restricción. Adoptar $p = 3$ produce además una fuerza restauradora más agresiva ante violaciones grandes, sin necesidad de inflar artificialmente $\beta$ para compensar la menor agresividad de una función cuadrática, manteniendo el modelo parsimonioso y los parámetros con interpretación física directa.

La constante $\beta$ (W·s⁻¹·°C⁻³) cuantifica el nivel de severidad del firmware ante el calor. Un fabricante conservador que prioriza la protección del silicio usará un $\beta$ alto; uno más permisivo, un $\beta$ bajo.

*Referencias: Ogata, K. — Ingeniería de control moderna, cap. Acciones básicas de control; Ames, A. D. et al. — Control Barrier Functions: Theory and Applications; Nocedal, J. & Wright, S. J. — Numerical Optimization; Rao, S. S. — Engineering Optimization: Theory and Practice; Brooks, D. & Martonosi, M. — Dynamic Thermal Management for High-Performance Microprocessors.*



## 2.2 Definición de Constantes


Hasta aquí el modelo contiene: $k_1$, $k_2$, $\gamma$, $\delta$, $\alpha$, $\beta$. Para garantizar que el modelo matemático refleje la realidad del hardware y permita simular intervenciones físicas, las constantes de las ecuaciones se derivan agrupando las velocidades tangibles y capacidades de los componentes. 
 

### 2.2.1 Variables Base de Hardware

Estas son las especificaciones del equipo que alimentan los parámetros del modelo:

| Símbolo | Unidad | Descripción |
|---------|--------|---|
| $N_c$ | — | Número de núcleos físicos del procesador |
| $N_h$ | — | Número de hilos lógicos (threads) |
| $f_p$ | GHz | Frecuencia de reloj del procesador |
| $C_p$ | MB | Tamaño de la caché del procesador |
| $v_{ram}$ | MT/s | Velocidad de la memoria RAM (megatransferencias por segundo) |
| $M_{ram}$ | GB | Capacidad total de RAM |
| $v_{alm}$ | MB/s | Velocidad de lectura/escritura del almacenamiento (NVMe/SSD) |
| $\Phi_{aire}$ | CFM | Tasa de flujo de aire de los ventiladores (pies cúbicos por minuto) |
| $H_d$ | W/°C | Eficiencia del disipador de calor (conductancia térmica) |
| $v_{bus}$ | GT/s | Velocidad del bus de la placa base (gigatransferencias por segundo) |



**Dinámica Térmica:**

- $k_1 = \kappa_0 \cdot (f_p \cdot N_c)$: Tasa de calentamiento (°C·s⁻¹·W⁻¹). A mayor frecuencia y más núcleos activos, más transistores conmutando simultáneamente y mayor disipación de energía como calor (Efecto Joule). El factor $\kappa_0$ captura la resistencia térmica específica del proceso litográfico del silicio (5 nm, 7 nm, etc.) y qué fracción de la energía eléctrica se convierte en calor en lugar de en cómputo útil. Equivale al inverso de la capacitancia térmica concentrada $C_{th}$ del modelo de Incropera: $k_1 \equiv \frac{1}{C_{th}}$.

- $k_2 = \eta_0 \cdot (\Phi_{aire} \cdot H_d)$: Coeficiente de disipación térmica (s⁻¹). La capacidad de enfriamiento depende del producto entre el flujo de aire que atraviesa el disipador ($\Phi_{aire}$, en CFM) y su conductancia térmica ($H_d$, en W/°C): mayor flujo y mejor disipador producen un $k_2$ más alto. El factor $\eta_0$ corrige la geometría real del chasis —obstrucciones en las rejillas, flujo no laminar, presión ambiental— y garantiza la consistencia dimensional. Equivale a $\frac{1}{R_{th} C_{th}}$ del modelo de capacitancia concentrada.

**Dinámica de Inferencia:**

- $\gamma = \gamma_0 \cdot (N_h \cdot f_p)$: Eficiencia computacional (tokens·s⁻²·W⁻¹). Basado en la Ley de Hierro del Rendimiento (*Iron Law*), el throughput bruto de cómputo es proporcional al número de hilos lógicos y a la frecuencia del reloj. El factor $\gamma_0$ representa las Instrucciones Por Ciclo (IPC) específicas para cálculo matricial — la operación nuclear de los LLMs, típicamente acelerada por la NPU — y cuantifica cuánta inferencia útil se extrae por cada julio de energía en esa arquitectura.

- $\delta = \frac{\delta_0}{\min(v_{ram},\, v_{bus})}$: Fricción por saturación de memoria (s⁻¹). Según el Modelo Roofline, la generación de tokens en LLMs opera en el régimen *memory bound*: el cuello de botella real no es la capacidad de cómputo sino el ancho de banda por el que los pesos del modelo llegan al procesador. El decaimiento del rendimiento es por tanto inversamente proporcional a la velocidad del enlace más lento en la ruta de datos principal, ya sea la RAM (MT/s) o el bus PCIe (GT/s). El factor $\delta_0$ cuantifica el overhead intrínseco del sistema operativo y el planificador de memoria.

**Control de Potencia:**

- $\alpha = \alpha_0 \cdot \frac{N_h}{N_c}$: Ganancia del gobernador del SO (W·s⁻¹·(token·s⁻¹)⁻¹). La ratio $N_h/N_c$ es el índice de hyperthreading: un procesador con hyperthreading activo expone más capacidad lógica al sistema operativo, induciendo al gobernador de energía a ser más agresivo en la asignación de potencia. El factor $\alpha_0$ representa la sensibilidad del perfil energético seleccionado: alto en "Máximo rendimiento", bajo en "Ahorro de batería".

- $\beta = \beta_0 \cdot f_p$: Factor de pánico del firmware (W·s⁻¹·°C⁻³). Los procesadores a mayor frecuencia son más vulnerables al daño térmico por migración de electrones, por lo que el firmware escala su respuesta de corte con $f_p$. El factor $\beta_0$ captura la filosofía de protección del fabricante: valores altos corresponden a fabricantes conservadores que priorizan la integridad del silicio; valores bajos, a perfiles orientados al rendimiento sostenido.
<!-- 

#### $k_1$ — Tasa de calentamiento (°C·s⁻¹·W⁻¹)

$$k_1 = \kappa_0 \cdot (f_p \cdot N_c)$$

La potencia entregada al procesador genera calor a una tasa proporcional a la frecuencia de operación y al número de núcleos activos (más núcleos a mayor frecuencia = más transistores commutando = más calor). El factor $\kappa_0$ (°C·s⁻¹·W⁻¹·GHz⁻¹·núcleo⁻¹) captura la resistencia térmica específica de la arquitectura del silicio (proceso litográfico de 5 nm, 7 nm, etc.) y qué fracción de la energía se convierte en calor en lugar de en cómputo.

**Rango típico de $\kappa_0$:** $0.001$ – $0.005$ °C·s⁻¹·W⁻¹·(GHz·núcleo)⁻¹ según proceso litográfico.

#### $k_2$ — Coeficiente de disipación térmica (s⁻¹)

$$k_2 = \eta_0 \cdot (\Phi_{aire} \cdot H_d)$$

La capacidad de disipación depende del producto entre el flujo de aire que atraviesa el disipador y la conductancia térmica de este. Mayor flujo de aire y mayor eficiencia del disipador producen un $k_2$ más alto (enfriamiento más rápido). El factor $\eta_0$ (s⁻¹·CFM⁻¹·(W/°C)⁻¹) corrige la geometría real del chasis del portátil: las rejillas de ventilación tienen obstrucciones, el aire no fluye de forma laminar perfecta, y la presión ambiental afecta la eficiencia del ventilador.

**Nota de dimensionalidad:** Las unidades de $\Phi_{aire} \cdot H_d$ son CFM·(W/°C). El factor $\eta_0$ absorbe la conversión a s⁻¹ de modo que $k_2$ tenga las unidades correctas.

**Rango típico de $\kappa_0$:** $0.002$ – $0.01$ s⁻¹·(CFM·W·°C⁻¹)⁻¹.

#### $\gamma$ — Eficiencia computacional (tokens·s⁻²·W⁻¹)

$$\gamma = \gamma_0 \cdot (N_h \cdot f_p) \cdot \ln\!\left(\frac{C_p \cdot v_{bus}}{C_0 \cdot v_0}\right)$$

La velocidad con que la potencia se convierte en inferencia útil depende del paralelismo ($N_h$ hilos a frecuencia $f_p$) y del ancho de banda de la jerarquía de memoria. El término logarítmico modela el **rendimiento marginal decreciente** del subsistema de memoria: duplicar la caché no duplica el rendimiento porque los accesos de los LLMs tienen patrones de localidad no uniforme.

**Corrección crítica de dimensionalidad:** El argumento del logaritmo debe ser adimensional. Se normalizan la caché y el bus respecto a valores de referencia:

$$C_0 = 1 \text{ MB}, \quad v_0 = 1 \text{ GT/s}$$

De este modo $\ln\!\left(\frac{C_p[\text{MB}] \cdot v_{bus}[\text{GT/s}]}{1 \text{ MB} \cdot 1 \text{ GT/s}}\right)$ es adimensional y matemáticamente correcto. El factor $\gamma_0$ tiene unidades de tokens·s⁻²·W⁻¹·(GHz·hilo)⁻¹ y representa la eficiencia IPC específica para cálculo matricial (operación nuclear de los LLMs, típicamente acelerada por la NPU).

#### $\delta$ — Fricción por saturación de memoria (s⁻¹)

$$\delta = \frac{\delta_0}{v_{ram} \cdot M_{ram} \cdot v_{alm}}$$

El denominador representa el "espacio de maniobra" del subsistema de memoria: más RAM, más rápida, con mejor almacenamiento secundario, reducen la fricción computacional. Cuando la RAM es escasa o lenta, el sistema debe hacer paginación frecuente hacia el almacenamiento, lo que incrementa el cuello de botella ($\delta$ más alto). El factor $\delta_0$ (s⁻¹·MT·s⁻¹·GB·MB·s⁻¹) absorbe las unidades y cuantifica el overhead intrínseco del sistema operativo y el planificador de memoria.

**Verificación de dimensionalidad:** $\delta$ debe tener unidades de s⁻¹ para que $\delta I(t)$ tenga unidades de tokens·s⁻², que es lo que necesita $\frac{dI}{dt}$.

#### $\alpha$ — Ganancia del gobernador de energía del SO (W·s⁻¹·(tokens/s)⁻¹)

$$\alpha = \alpha_0 \cdot \frac{N_h}{N_c}$$

La ratio $N_h / N_c$ es el índice de hiperthreading del procesador: cuántos hilos lógicos existen por núcleo físico. Un procesador con hyperthreading activo (ratio $= 2$) expone más capacidad lógica al SO, lo que induce al gobernador de energía a ser más agresivo en la asignación de potencia. El factor $\alpha_0$ representa la sensibilidad intrínseca del perfil de energía: en modo "Máximo Rendimiento" ($\alpha_0$ alto), el gobernador reacciona rápidamente al déficit de inferencia; en "Equilibrado" o "Ahorro de batería" ($\alpha_0$ bajo), la respuesta es lenta y conservadora.

#### $\beta$ — Factor de pánico del firmware (W·s⁻¹·°C⁻³)

$$\beta = \beta_0 \cdot f_p$$

Los procesadores que operan a mayor frecuencia de reloj son más sensibles a los picos térmicos (el daño por migración de electrones es más rápido a frecuencias altas). Por tanto, el firmware de fabricantes como Intel o AMD calibra su respuesta térmica en función de la frecuencia: a mayor $f_p$, mayor urgencia en el corte de potencia ante superaciones de $T_{crit}$. El factor $\beta_0$ (W·s⁻¹·°C⁻³·GHz⁻¹) captura la filosofía de diseño del fabricante respecto a la protección del silicio: fabricantes conservadores usan $\beta_0$ alto (corte agresivo), fabricantes orientados al rendimiento usan $\beta_0$ más bajo.

La Justificación Teórica de Constantes

 Para defender tu modelo en la presentación, te basarás en dos grandes pilares teóricos:

    Para el Rendimiento ($\gamma$ y $\delta$): El Modelo Roofline y la Ley de HierroLa "Ley de Hierro del Rendimiento de Procesadores" (Iron Law of Processor Performance) establece que el tiempo de ejecución depende de la cantidad de instrucciones, los ciclos por instrucción (CPI) y el tiempo del ciclo de reloj. El inverso del CPI es el IPC (Instrucciones Por Ciclo). Tu constante $\gamma$ representa esta capacidad bruta de cómputo.Por otro lado, el Modelo Roofline es el estándar de oro en la academia para evaluar algoritmos de IA. Este modelo establece que un procesador está limitado por dos "techos": su capacidad máxima de cálculo (Compute Bound) o su ancho de banda de memoria (Memory Bound). La generación de tokens en modelos de lenguaje (decodificación autorregresiva) está severamente limitada por el ancho de banda de la memoria. Tu constante $\delta$ (la fricción) modela exactamente este cuello de botella.

    Para la Disipación ($k_1$ y $k_2$): Resistencia TérmicaLa capacidad de un disipador para extraer calor depende del flujo másico de aire. La disipación térmica de un equipo cambia dinámicamente dependiendo del flujo de aire disponible (medido en CFM, pies cúbicos por minuto). La ecuación física establece que la cantidad de disipación de calor es directamente proporcional a los CFM y a la diferencia de temperatura.

-->


## 2.3 Análisis Dimensional y Balance del Sistema Dinámico

Para garantizar la viabilidad de la integración numérica mediante el método de Runge-Kutta de cuarto orden (RK4), se exige que el modelo sea dimensionalmente homogéneo. El tamaño de paso de integración $h$ se define en segundos (**s**). A continuación, se demuestra el balance estricto de unidades para las tres ecuaciones diferenciales acopladas.

Las variables de estado del sistema tienen las siguientes unidades físicas:
* Temperatura $T(t)$: [**°C**]
* Inferencia $I(t)$: [**tokens/s**]
* Potencia $P(t)$: [**W**]

#### 1. Ecuación de Dinámica Térmica
La tasa de cambio de la temperatura debe resultar estrictamente en grados Celsius por segundo (**°C/s**).
$$\frac{dT}{dt} = k_1 P(t) - k_2 (T(t) - T_{amb})$$

* **Término de calentamiento:** $k_1$ debe convertir vatios a una tasa de temperatura. Sus unidades son [**°C/(W·s)**] o [**°C/J**].
  $$\left[\frac{^\circ\text{C}}{\text{W}\cdot\text{s}}\right] \cdot [\text{W}] = \left[\frac{^\circ\text{C}}{\text{s}}\right]$$
* **Término de disipación:** La diferencia de temperatura ya está en grados, por lo que $k_2$ debe ser una tasa temporal pura [**1/s**].
  $$\left[\frac{1}{\text{s}}\right] \cdot [^\circ\text{C}] = \left[\frac{^\circ\text{C}}{\text{s}}\right]$$

#### 2. Ecuación de Dinámica de Inferencia
La tasa de aceleración del procesamiento debe resultar en [**tokens/s²**].
$$\frac{dI}{dt} = \gamma P(t) - \delta I(t)$$

* **Término de aceleración:** $\gamma$ convierte la potencia eléctrica en aceleración de cómputo. Sus unidades son [**tokens/(W·s²)**].
  $$\left[\frac{\text{tokens}}{\text{W}\cdot\text{s}^2}\right] \cdot [\text{W}] = \left[\frac{\text{tokens}}{\text{s}^2}\right]$$
* **Término de fricción/saturación:** La inferencia ya es una velocidad. $\delta$ actúa como una constante de decaimiento temporal [**1/s**].
  $$\left[\frac{1}{\text{s}}\right] \cdot \left[\frac{\text{tokens}}{\text{s}}\right] = \left[\frac{\text{tokens}}{\text{s}^2}\right]$$

#### 3. Ecuación de Control de Potencia
La tasa de cambio en la asignación de potencia debe resultar en [**W/s**].
$$\frac{dP}{dt} = \alpha (I_{obj} - I(t)) - \beta \max(0, T(t) - T_{crit})^3$$

* **Término del Gobernador (SO):** El error de inferencia está en **tokens/s**. $\alpha$ es la ganancia de potencia por cada token de déficit, con unidades [**W/tokens**].
  $$\left[\frac{\text{W}}{\text{tokens}}\right] \cdot \left[\frac{\text{tokens}}{\text{s}}\right] = \left[\frac{\text{W}}{\text{s}}\right]$$
* **Término de Función Barrera (Firmware):** El exceso de temperatura está elevado al cubo, resultando en **°C³**. $\beta$ debe castigar este volumen térmico convirtiéndolo en una caída de potencia por segundo, con unidades [**W/(s·°C³)**].
  $$\left[\frac{\text{W}}{\text{s}\cdot^\circ\text{C}^3}\right] \cdot [^\circ\text{C}^3] = \left[\frac{\text{W}}{\text{s}}\right]$$

 


## 2.4 Condiciones Iniciales

Las condiciones iniciales representan el estado del equipo en $t = 0$, justo antes de lanzar el proceso de inferencia (equipo encendido pero en reposo):

| Variable | Valor inicial | Significado físico |
|----------|:---:|---|
| $T(0)$ | $45$ °C | Temperatura de reposo: el procesador ya tiene carga base del SO (no está frío) |
| $P(0)$ | $15$ W | Potencia base: consumo típico del SO en inactividad |
| $I(0)$ | $0$ tokens/s | Inferencia detenida: el LLM aún no ha iniciado |

**Parámetros fijos del entorno:**

| Parámetro | Valor | Descripción |
|-----------|:---:|---|
| $T_{amb}$ | $25$ °C | Temperatura ambiente (sala estándar) |
| $T_{crit}$ | $90$ °C | Umbral de activación del throttling (especificación del fabricante) |
| $I_{obj}$ | $20$ tokens/s | Velocidad objetivo del gobernador del SO |
| $I_{min}$ | $5$ tokens/s | Umbral mínimo de viabilidad operativa |
| $P_{min}$ | $5$ W | Potencia mínima física del procesador |

Para que el modelo sea simulable, se necesitan valores numéricos concretos. Se usa como referencia un portátil de gama media-alta típico para IA local (especificaciones aproximadas de un equipo con procesador de 8 núcleos a 3.5 GHz):

| Variable de hardware | Símbolo | Valor de referencia | Unidad |
|---|---|:---:|---|
| Núcleos físicos | $N_c$ | 8 | — |
| Hilos lógicos | $N_h$ | 16 | — |
| Frecuencia del procesador | $f_p$ | 3.5 | GHz |
| Caché del procesador | $C_p$ | 24 | MB |
| Velocidad RAM | $v_{ram}$ | 4800 | MT/s |
| Capacidad RAM | $M_{ram}$ | 32 | GB |
| Velocidad almacenamiento | $v_{alm}$ | 3500 | MB/s |
| Flujo de aire | $\Phi_{aire}$ | 2.5 | CFM |
| Eficiencia del disipador | $H_d$ | 5.0 | W/°C |
| Velocidad del bus | $v_{bus}$ | 16 | GT/s |

| Factor de escala | Símbolo | Valor inicial | Justificación |
|---|---|:---:|---|
| Ineficiencia térmica del silicio | $\kappa_0$ | $0.002$ | Proceso de 7 nm, resistencia moderada |
| Constante de convección | $\eta_0$ | $0.004$ | Chasis con rejillas parcialmente obstruidas |
| Eficiencia IPC para IA | $\gamma_0$ | $0.015$ | NPU habilitada, cálculo matricial eficiente |
| Factor de latencia intrínseca | $\delta_0$ | $1.2 \times 10^{8}$ | SO con overhead estándar; absorbe unidades |
| Ganancia del gobernador del SO | $\alpha_0$ | $0.8$ | Perfil "Máximo Rendimiento" |
| Factor de pánico del firmware | $\beta_0$ | $0.0001$ | Fabricante con protección moderada |

Con estos valores, los parámetros del modelo resultan:

$$k_1 = 0.002 \times (3.5 \times 8) = 0.056 \text{ °C*s}^{-1}\text{*W}^{-1}$$

$$k_2 = 0.004 \times (2.5 \times 5.0) = 0.050 \text{ s}^{-1}$$

$$\gamma = 0.015 \times (16 \times 3.5) \times \ln\!\left(\frac{24 \times 16}{1}\right) \approx 0.015 \times 56 \times 5.99 \approx 5.03 \text{ tokens*s}^{-2}\text{*W}^{-1}$$

$$\delta = \frac{1.2 \times 10^{8}}{4800 \times 32 \times 3500} \approx \frac{1.2 \times 10^{8}}{5.376 \times 10^{8}} \approx 0.223 \text{ s}^{-1}$$

$$\alpha = 0.8 \times \frac{16}{8} = 1.6 \text{ W*s}^{-1}\text{*(tokens/s)}^{-1}$$

$$\beta = 0.0001 \times 3.5 = 3.5 \times 10^{-4} \text{ W*s}^{-1}\text{*°C}^{-3}$$

---

## 3. Análisis de Puntos de Equilibrio

El análisis de equilibrio responde directamente a la pregunta central del proyecto: ¿puede el sistema estabilizarse con rendimiento viable?

### 3.1 Equilibrio en el Régimen Sin Throttling ($T^* < T_{crit}$)

Cuando la temperatura no supera el umbral crítico, el término de throttling es cero. Igualando las tres derivadas a cero simultáneamente:

$$\frac{dT}{dt} = 0 \implies k_1 P^* = k_2(T^* - T_{amb}) \implies T^* = T_{amb} + \frac{k_1}{k_2} P^*$$

$$\frac{dI}{dt} = 0 \implies I^* = \frac{\gamma}{\delta} P^*$$

$$\frac{dP}{dt} = 0 \implies \alpha(I_{obj} - I^*) = 0 \implies I^* = I_{obj}$$

De la tercera ecuación se obtiene $I^* = I_{obj}$. Sustituyendo en la segunda:

$$P^* = \frac{\delta \cdot I_{obj}}{\gamma}$$

Y en la primera:

$$T^* = T_{amb} + \frac{k_1 \delta \cdot I_{obj}}{k_2 \gamma}$$

Con los valores numéricos de referencia:

$$P^* = \frac{0.223 \times 20}{5.03} \approx 0.887 \text{ W} \quad \text{(revisión: este valor indica que } \gamma \text{ necesita recalibración)}$$

> **Nota de calibración:** Si $P^*$ resulta inconsistente con el rango físico esperado (15–45 W para un portátil bajo carga), es señal de que los factores de escala necesitan ajuste iterativo. Esto es parte del proceso de calibración del modelo y debe documentarse en el informe como parte de la metodología.

La condición de existencia del equilibrio estable en este régimen es:

$$T^* = T_{amb} + \frac{k_1 \delta \cdot I_{obj}}{k_2 \gamma} < T_{crit}$$

$$\iff \frac{k_1 \delta}{k_2 \gamma} < \frac{T_{crit} - T_{amb}}{I_{obj}}$$

Esta desigualdad es la **condición de viabilidad** del equipo para sostener la carga de IA sin throttling. Cuando se cumple, el sistema puede alcanzar un punto de equilibrio estable con inferencia plena. Cuando no se cumple, el equilibrio requiere que el throttling esté activo, y el sistema entra en régimen oscilatorio.

### 3.2 Estabilidad del Punto de Equilibrio — Análisis por Retroalimentación Negativa

Para verificar que el punto de equilibrio $(T^*, I^*, P^*)$ es un **atractor estable** — es decir, que el sistema regresa a él tras una pequeña perturbación — se analiza la dirección de las derivadas cuando el estado se desvía del equilibrio. Este argumento se basa directamente en la estructura física de las ecuaciones, sin requerir herramientas de álgebra matricial.

**Perturbación térmica.** Suponga que en un instante el sistema se encuentra en equilibrio pero una rafaga externa eleva la temperatura a $T^* + \varepsilon$ con $\varepsilon > 0$ pequeño. La ecuación térmica vale:

$$\frac{dT}{dt}\bigg|_{T^*+\varepsilon} = k_1 P^* - k_2\bigl((T^*+\varepsilon) - T_{amb}\bigr) = \underbrace{k_1 P^* - k_2(T^*-T_{amb})}_{=\,0 \text{ en equilibrio}} - k_2\varepsilon = -k_2\varepsilon < 0$$

La derivada se vuelve negativa: el sistema enfría activamente hacia $T^*$. El coeficiente $k_2 > 0$ actúa como una fuerza de amortiguación proporcional al desvío — exactamente la retroalimentación negativa que garantiza la devolución al equilibrio.

**Perturbación de inferencia.** Si $I$ cae por debajo de $I^*$, el término $\alpha(I_{obj} - I)$ se vuelve positivo, aumentando $\dot{P}$, lo que empuja $\dot{I} = \gamma P - \delta I$ hacia valores positivos, recuperando la velocidad de inferencia. El término $-\delta I(t)$ actúa simultáneamente como fricción que impide sobrepasar $I^*$.

**Condición necesaria para la estabilidad.** El argumento anterior funciona siempre que las constantes de amortiguación dominen sobre los términos de impulso, lo que se reduce a la misma condición de viabilidad derivada en §5.1:

$$\frac{k_1 \delta}{k_2 \gamma} < \frac{T_{crit} - T_{amb}}{I_{obj}}$$

Cuando esta desigualdad se satisface, cada desvío genera fuerzas restauradoras en las tres ecuaciones simultáneamente, y el punto de equilibrio es estable. Cuando no se satisface, las fuerzas restauradoras son insuficientes y el sistema entra en el régimen oscilatorio descrito en §5.3.

### 3.3 Régimen Oscilatorio — Cuando No Hay Equilibrio Estable

Cuando la condición de viabilidad no se cumple, el sistema entra en un ciclo repetitivo:

1. El SO aumenta la potencia buscando alcanzar $I_{obj}$.
2. La temperatura sube y supera $T_{crit}$.
3. El firmware activa el throttling cúbico, que colapsa la potencia.
4. La temperatura baja al perder la fuente de calor.
5. El SO intenta recuperar la potencia y el ciclo reinicia.

Este régimen es el que hace al equipo inviable para cargas de IA: la inferencia oscila entre valores altos (inestables térmicamente) y valores bajos (en recuperación post-throttling), sin converger nunca a un estado estable con $I \geq I_{min}$.

---

## 4. Cuatro Escenarios de Intervención

Los escenarios modifican los parámetros del hardware para evaluar qué intervenciones permiten mover el sistema desde el régimen oscilatorio hacia un equilibrio estable.

### Escenario 0 — Base (Control)

Portátil estándar sin modificaciones. Parámetros de la Sección 4.3. Este escenario puede o no tener equilibrio estable según la condición de viabilidad. Es el punto de referencia para medir variaciones porcentuales.

### Escenario 1 — Intervención de Software (Undervolting / Perfil Conservador)

**Modificación:** Reducción de $\alpha_0$ (gobernador menos agresivo) y limitación de $P_{max}$ a 35 W.

**Efecto esperado:** El SO sube la potencia más lentamente, dando tiempo al sistema de disipación a mantenerse por debajo de $T_{crit}$. Puede estabilizar el sistema a costa de reducir $I^*$ por debajo de $I_{obj}$.

**Variable de comparación:** ¿Alcanza $I^* \geq I_{min} = 5$ tokens/s en equilibrio?

### Escenario 2 — Intervención de Hardware Ligera (Base Refrigerante / Limpieza)

**Modificación:** Aumento de $\Phi_{aire}$ (más flujo de aire) → $k_2$ sube.

**Efecto esperado:** Mayor disipación térmica desplaza $T^*$ hacia abajo, permitiendo que el equilibrio sin throttling sea alcanzable. El sistema puede sostener $I_{obj}$ sin necesidad de sacrificar velocidad.

**Variable de comparación:** Variación porcentual de $T^*$ y de la frecuencia de oscilación respecto al escenario base.

### Escenario 3 — Intervención de Hardware Profunda (Pasta Térmica de Metal Líquido)

**Modificación:** Aumento de $H_d$ (conductancia del disipador) y ajuste de $\eta_0$ → $k_2$ sube sustancialmente.

**Efecto esperado:** La disipación térmica aumenta de forma significativa. Si el nuevo $k_2$ cumple la condición de viabilidad, el sistema converge al equilibrio estable con inferencia plena.

**Variable de comparación:** Tiempo de convergencia al equilibrio y máxima temperatura alcanzada durante la transición.

---

## 5. Metodología Numérica

### 5.1 Implementación de RK4

Para resolver el sistema vectorial $\dot{\mathbf{x}} = \mathbf{f}(\mathbf{x}, t)$ con $\mathbf{x} = (T, I, P)^T$, se implementa RK4 desde cero:

$$\mathbf{k}_1 = \mathbf{f}(\mathbf{x}_n, t_n)$$
$$\mathbf{k}_2 = \mathbf{f}\!\left(\mathbf{x}_n + \frac{h}{2}\mathbf{k}_1,\, t_n + \frac{h}{2}\right)$$
$$\mathbf{k}_3 = \mathbf{f}\!\left(\mathbf{x}_n + \frac{h}{2}\mathbf{k}_2,\, t_n + \frac{h}{2}\right)$$
$$\mathbf{k}_4 = \mathbf{f}(\mathbf{x}_n + h\,\mathbf{k}_3,\, t_n + h)$$
$$\mathbf{x}_{n+1} = \mathbf{x}_n + \frac{h}{6}(\mathbf{k}_1 + 2\mathbf{k}_2 + 2\mathbf{k}_3 + \mathbf{k}_4)$$

Después de cada paso, se aplica la siguiente condición de frontera programática sobre $P$:


#### Condición de Frontera Sobre $P(t)$

La ecuación planteada permite que $P(t)$ se vuelva negativa si el throttling es muy agresivo. Pero en la realidad un procesador siempre consume una potencia mínima en reposo $P_{min}$, por ejemplo el procesador mantiene sus relojes, el SO consume recursos base de conexion, etc.

Al programar el integrador numérico desde cero, es completamente válido —y necesario— incluir reglas lógicas que descarten resultados matemáticamente válidos pero físicamente imposibles. Al finalizar cada paso de integración, se aplica la siguiente corrección antes de avanzar al siguiente instante:

$$P_{n+1} \leftarrow \max\bigl(P_{min},\, P_{n+1}\bigr) \quad \text{con } P_{min} = 5 \text{ W}$$

Esta corrección **no modifica la ODE**; es una condición de frontera aplicada en el código del solucionador. Su efecto es equivalente a decirle al integrador: "si el resultado numérico de este paso arroja una potencia negativa o sub-mínima, trunca al valor físico mínimo admisible y continúa desde ahí". Garantiza que el modelo permanezca en un dominio físicamente admisible durante toda la simulación y debe documentarse explícitamente en la sección de implementación del informe.


### 5.2 Inestabilidad Inducida por Gradientes Extremos

El método RK4 es un integrador *explícito*: en cada paso calcula cuatro pendientes locales de la función $\mathbf{f}(\mathbf{x}, t)$ y las combina para extrapolar el estado al siguiente instante. Esta estrategia funciona bien cuando las pendientes cambian suavemente entre $t_n$ y $t_{n+1}$.

El problema surge cuando se activa el throttling. El término $-\beta\max(0, T - T_{crit})^3$ genera una derivada respecto a $T$ igual a $-3\beta(T - T_{crit})^2$, que crece con el *cuadrado* del exceso térmico. Esto significa que ante una superación de apenas $10\,°C$ sobre $T_{crit}$, la pendiente de $\dot{P}$ respecto a $T$ se vuelve pronunciada; ante una superación de $20\,°C$, esa pendiente se cuadruplica. El gradiente no es suave: **explota en función del estado del sistema**.

Cuando el tamaño de paso $h$ es demasiado grande, RK4 evalúa las pendientes en puntos intermedios que ya se encuentran dentro de la zona de gradiente extremo, y la extrapolación sobreestima masivamente el corte de potencia. Esto produce que $P_{n+1}$ caiga a valores físicamente absurdos en un solo paso, lo que a su vez colapsa $I$ y dispara $\dot{P}$ en el sentido contrario con igual violencia. El método entra en una oscilación numérica divergente que no refleja el comportamiento físico real — es un artefacto del integrador, no del sistema.

La solución es reducir $h$ lo suficiente para que, entre $t_n$ y $t_{n+1}$, la temperatura no avance lo necesario como para que el gradiente cambie de orden de magnitud. En el análisis de error se debe mostrar:

- El $h_{crítico}$ a partir del cual la solución RK4 diverge durante episodios de throttling activo.
- La convergencia para $h$ suficientemente pequeño, verificando que el error escala con $h^4$ en el régimen sin throttling.
- La comparación con RK45, cuyo control de paso adaptativo reduce $h$ automáticamente al detectar cambios abruptos de pendiente, obteniendo la misma solución con menor costo computacional en el régimen libre.

### 5.3 Análisis de Error y Convergencia

Para cada valor de $h \in \{1.0, 0.5, 0.1, 0.05, 0.01, 0.005\}$ segundos, se calcula el error relativo global respecto a RK45:

$$E_{rel}(h) = \max_{t \in [0, T_{sim}]} \frac{\|\mathbf{x}_{RK4}(t; h) - \mathbf{x}_{RK45}(t)\|}{\|\mathbf{x}_{RK45}(t)\|}$$

Se espera convergencia de orden 4 en el régimen suave ($E \sim h^4$) y degradación del orden en la vecindad de $T = T_{crit}$ (por la no diferenciabilidad de $\max$). Esta degradación es un resultado de análisis legítimo y debe discutirse en el informe.

---

## 6. Síntesis: La Narrativa del Proyecto

El hilo conductor del proyecto de principio a fin es:

1. **Motivación:** Los portátiles modernos no pueden sostener cargas de IA de forma indefinida sin colapsar térmicamente.
2. **Modelado:** El fenómeno se describe mediante tres ecuaciones acopladas cuya no-linealidad impide solución analítica.
3. **Pregunta:** ¿Existe un punto de equilibrio viable ($T^* < T_{crit}$, $I^* \geq I_{min}$)? ¿Cuándo sí y cuándo no?
4. **Método:** RK4 programado desde cero, con análisis de convergencia y validación contra RK45.
5. **Comparación:** Cuatro escenarios que modifican parámetros físicos y comparan si el equilibrio es alcanzable.
6. **Conclusión:** Identificación de la condición de viabilidad y qué intervención (software o hardware) es más efectiva para cruzar ese umbral.

Cada componente del proyecto (animación, análisis de fases, sensibilidad a condiciones iniciales, puntos de equilibrio) debe conectarse con esta narrativa central. La animación, por ejemplo, debe visualizar la trayectoria del vector de estado $(T, I, P)$ en el espacio de fases, mostrando si converge a un punto o queda atrapada en un ciclo.


