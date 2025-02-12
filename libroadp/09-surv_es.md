# Modelos de supervivencia {#surv}

Francisco Urdinez^[E-mail: furdinez\@uc.cl]

### Lecturas sugeridas {-}

- Allison, P. D. (2014). *Event history and survival analysis* (2nd ed.). Thousand Oaks: SAGE.

- Box-Steffensmeier, J. M. & Jones, B. S. (2004). *Event history modeling: A guide for social scientists*. Cambridge: Cambridge University Press.

- Broström, G. (2018). *Event history analysis with R*. Boca Raton: CRC Press.

- Golub, J. (2008). Discrete Choice Methods. In J. M. Box-Steffensmeier, H. E. Brady, & D. Collier (Eds.), *The Oxford Handbook of Political Methodology* (pp. 530–546). Oxford: Oxford University Press.

### Los paquetes que necesitas instalar {-}

- `tidyverse` [@R-tidyverse], `paqueteadp` [@R-paqueteadp], `skimr` [@R-skimr], `countrycode` [@R-countrycode], `ggalt` [@R-ggalt], `survival` [@R-survival], `survminer` [@R-survminer], `texreg` [@R-texreg].

## Introducción

Hay una serie de cuestiones recurrentes en relación con el análisis de datos políticos que aún no hemos abordado. En muchas ocasiones estamos interesados en saber por qué ciertos eventos duran lo que duran, o por qué tardan más que otros en ocurrir. ¿Por qué la paz es tan duradera en ciertos países, mientras que otros están en guerra constante? ¿Cuál era la posibilidad de que se produjeran disturbios sociales en Venezuela en 2018?  ¿Por qué algunos legisladores permanecen en el cargo durante muchos períodos consecutivos, mientras que otros ni siquiera son reelegidos una vez?  ¿Cuánto tiempo tarda un sindicato en hacer huelga durante una crisis económica?

Todas estas preguntas son sobre la duración de un evento. El momento de ocurrencia de un evento es parte de la respuesta que buscamos, necesitamos de un modelo que nos permita llegar a esta respuesta [@box-steffensmeierEventHistoryModeling2004], una de las principales referencias en Ciencia Política de este método, se refiere a ellos como “modelos de eventos históricos” aunque buena parte de la literatura los llama modelos de supervivencia o modelos de duración. Si bien en la Ciencia Política no son modelos tan utilizados como uno creería (en el fondo, casi todas las preguntas que nos hacemos pueden ser reformuladas en una pregunta sobre la duración del evento), las ciencias médicas han explorado estos métodos en profundidad, y muchas las referencias que uno encuentra en `R` sobre paquetes accesorios a estos modelos son de departamentos bioestadísticos y médicos. De allí que “modelos de supervivencia” sea el nombre más frecuentemente utilizado para estos modelos, ya que en medicina comenzó a utilizárselos para modelar qué variables afectaban la sobrevida de sus pacientes enfermos.  

Podemos tener dos tipos de bases de datos para estos problemas. Por un lado, podemos tener una base en formato de panel en el que para un momento dado nuestra variable dependiente codifica si el evento ha ocurrido (`=1`) o no (`=0`). Así, por ejemplo, podemos tener una muestra de veinte países para cincuenta años (1965-2015) en los que nuestra variable de interés es si el país ha implementado una reforma constitucional. La variable independiente asumirá el valor 1 para el año [1994 en Argentina](https://es.wikipedia.org/wiki/Reforma_constitucional_argentina_de_1994), pero será 0 para el resto de los años en este país. Por otro lado, podemos tener una base de datos transversal en la que cada observación aparece codificada apenas una vez. En este caso necesitamos, además de la variable que nos dirá si en el periodo de interés el evento ocurrió o no para cada observación (por ejemplo, Argentina debería ser codificada como “1”), una variable extra que codifique el tiempo de “supervivencia” de cada observación, es decir, cuánto tiempo pasó hasta que finalmente el evento sucedió. Para el caso de Argentina, esta variable codificará 29 (años), que es lo que demoró en implementarse una reforma constitucional desde 1965. La elección del año de partida, como podrá sospechar, es decisión del investigador, pero tiene un efecto enorme sobre nuestros resultados. Además, muchas veces la fecha de inicio acaba determinada por la disponibilidad de datos y se alejan del ideal que quisiéramos modelar. 

Supongamos que nos hacemos la pregunta que se hizo David Altman [-@altmanCitizenshipContemporaryDirect2019] “¿Por qué algunos países demoran menos que otros en implementar instancias de democracia directa?”. Para ello tenemos una base de datos en formato de panel que parte del año 1900 y que llega a 2016 para 202 países (algunas observaciones, como la Unión Soviética se transforman en otras observaciones a partir de un determinado año en que dejan de existir). Al observar sus datos uno nota algo que probablemente también te suceda en tu base de datos. Para el año 2016 apenas un pequeño porcentaje de países había implementado este tipo de mecanismos (27% para ser más precisos) pero la base está censurada ya que a partir de ese año no sabemos que ha ocurrido con los países que aún no habían implementado mecanismos de democracia directa. No todas las observaciones han “muerto” aún, ¿cómo saber cuándo lo harán? Ésta es una pregunta válida, que podremos responder con este tipo de modelos, ya que podemos calcular el tiempo que demorará cada uno de los países censurados en nuestra muestra (con la información que le damos al modelo, que siempre es incompleta). 

En nuestra base de datos tendremos, al menos, cuatro tipos de observaciones (ver figura \@ref(fig:surv-ex-obs)): 

- (a) aquellas que, para el momento en que tenemos datos ya estaban en la muestra, aunque no siempre sabremos hace cuanto que “existen”. Son, en la figura, las observaciones B y C. En la base de datos de Altman, por ejemplo, México ya existía como entidad política en 1900, cuando su base de datos parte (sabemos que la Primera República Federal existió como entidad política desde octubre de 1824, por lo que México sería codificado como existente a partir de esa fecha). También sabemos que en 2012, por primera vez, México implementó una iniciativa de democracia directa, lo que define como positiva la ocurrencia del evento que nos interesa medir. Así, México sería como la observación B de la figura.

- (b) Algunas observaciones estarán desde el comienzo de la muestra, y existirán hasta el último momento sin haber registrado el evento de interés. Tal es el caso, de la observación C en la figura. En la muestra de Altman un ejemplo sería Argentina, que desde 1900 está registrado en la base (ya había "nacido"), y hasta el último año de la muestra no había registrado instancias de democracia directa (no "murió"), lo que la transforma en una observación censurada.

Por razones prácticas, no cambia saber qué ocurrió a partir del año en que nuestra base termina. Por ejemplo, en la figura, nuestra base cubre hast $t_7$, y sabemos que en $t_8$ la observación C aún no había muerto, y la observación D lo había hecho en $t_8$. En nuestra base, C y D serán ambas observaciones censuradas en $t_7$.

- (c) Algunas observaciones pueden entrar “tarde” en la muestra, como es el caso de las observaciones A y D. Por ejemplo, Eslovenia entra a la muestra de Altman en 1991, que es cuando se independiza de Yugoslavia y "nace" como país.

- (d) Algunas observaciones, independientemente de cuando entren a la muestra, "moriran" durante el periodo analizado. Por ejemplo, A y B mueren dentro del periodo que hemos medido entrte $t_1$ y $t_7$. Ya para la observación D, no registramos su muerte. Hay un caso no considerado en el ejemplo, de observaciones que nacen y mueren sucesivamente a lo largo del periodo de estudio. Para ellas, deberemos decidir si las tratamos como observaciones independientes, o si modelamos la posibilidad de morir más de una vez. Si es así, la probabilidad de morir por segunda vez deberá estar condicionada por la probabilidad de haber muerto (y cuando!) por primera vez. Este es un tipo de caso algo más complejo que no cubriremos en este capítulo.


<div class="figure" style="text-align: center">
<img src="00-images/surv/ex_obs.png" alt=" Ejemplo de observaciones presentes en una base de datos de supervivencia " width="70%" />
<p class="caption">(\#fig:surv-ex-obs) Ejemplo de observaciones presentes en una base de datos de supervivencia </p>
</div>

## ¿Cómo interpretamos las tasas de riesgo?
Los modelos de supervivencia se interpretan a partir de la probabilidad de que en un momento dado el evento de interés ocurra siendo que no ha ocurrido aún. Esta probabilidad recibe el nombre de tasa de riesgo o *Hazard rate*. Partimos sabiendo que tenemos una variable, que llamaremos $T$, y que representa un valor aleatorio positivo y que tiene una distribución de probabilidades (correspondiente a la probabilidad del evento ocurrir en cada uno de los momentos posibles) que llamaremos $f(t)$. Esta probabilidad se puede expresar de manera acumulada, como una densidad acumulada $F(t)$. Como se expresa en la siguiente formula, $F(t)$ viene dada por la probabilidad de que el tiempo de supervivencia $T$ sea menor o igual a un tiempo específico $t$ :

$$F(t)=\int\limits_0^t f(u)d(u)=Pr(T)\leq t)$$

La función de supervivencia $\hat S(t)$, que es un concepto clave en estos modelos, está relacionado con $F(t)$, ya que

$$\hat S(t)= 1-F(t)=Pr(T\geq t)$$

En otras palabras, la función de supervivencia es la probabilidad inversa de $F(t)$, pues dice respecto a la probabilidad de que el tiempo de supervivencia $T$ sea mayor o igual un tiempo $t$ de interés. Para el ejemplo concreto de Altman, uno podría preguntarse cuál es la probabilidad de que un país no implemente un mecanismo de democracia directa (lo que sería equivalente a “sobrevivir” a dicha implementación) siendo que ya ha sobrevivido a los mismos por 30 años. A medida que más y más países en la muestra van implementando iniciativas de democracia directa, la probabilidad de supervivencia va disminuyendo. 

Los coeficientes de los modelos de supervivencia se suelen interpretar como una tasa de riesgo, que es el cociente de la probabilidad de que el evento suceda, y su función de supervivencia

$$h(t)=\frac{f(t)}{S(t)}$$

Así, la tasa de riesgo indica la tasa a la que las observaciones “mueren” en nuestra muestra en el momento $t$, considerando que la observación ha sobrevivido hasta el momento $t$. Veremos más adelante cómo en el ejemplo de Altman podemos interpretar los coeficientes de nuestras regresiones como tasas de riesgo. En definitiva, la tasa de riesgo $h(t)$ es el riesgo de que el evento ocurra en un intervalo de tiempo determinado, que viene dado por 

$$f(t)=\lim_{\bigtriangleup x \to 0} \frac {P(t+\bigtriangleup t > T \geq t)}{\bigtriangleup t}$$

## El modelo Cox de riesgos proporcionales

Hay dos tipos de modelos de supervivencia, los llamados modelos paramétricos y los llamados semi-parametricos. Los primeros son aquellos que hacen supuestos sobre las características de la población a la que la muestra pertenece. En este caso, los supuestos son sobre el riesgo de base o “baseline hazard”, es decir, sobre el riesgo de que el evento ocurra cuando todas nuestras variables independientes sean iguales a cero. El tipo de modelo de supervivencia más común para esta categoría es el modelo de Weibull. Por otro lado, los modelos semi-parametricos no hacen ningún tipo de supuestos sobre la función de base, ya que ésta es estimada a partir de los datos. El ejemplo más famoso de esta especificación es la del modelo de Cox. 

El *Oxford Handbook* sobre metodología política dedica un [capítulo entero](https://www.oxfordhandbooks.com/view/10.1093/oxfordhb/9780199286546.001.0001/oxfordhb-9780199286546-e-23) a discutir modelos de supervivencia, y en él se toma una posición fuerte en favor de los modelos semi-parametricos. Aquí seguiremos dicha recomendación ya que las ventajas son varias. Por un lado, como no se hacen presupustos sobre la función del riesgo de base, su estimación es mucho más precisa. En una estimación paramétrica, elegir un “baseline hazard” equivocado siginificará que todo nuestro trabajo analítico estará sesgado. La decisión de la forma que adopta la curva de base en un modelo de Weibull debería estar orientado por razones teóricas de cuál es el efecto de nuestra variable independiente sobre la probabilidad de supervivencia de la observación (ver figura \@ref(fig:surv-hazards)). Sin embargo, no siempre nuestra teoría define tales presupuestos. Elegir una especificación por Cox nos ahorra tomar una decisión tan costosa.

<div class="figure" style="text-align: center">
<img src="00-images/surv/hazards.png" alt=" Los diferentes peligros de la línea de base en el modelo de Weibull " width="80%" />
<p class="caption">(\#fig:surv-hazards) Los diferentes peligros de la línea de base en el modelo de Weibull </p>
</div>

Una segunda ventaja de los modelos semi-parametricos sobre los paramétricos tiene que ver con el presupuesto de riesgos proporcionales. Ambos, modelos paramétricos y semi-parametricos, asumen que los riesgos entre dos individuos cualquiera de la muestra se mantienen constantes a lo largo de todo su periodo de supervivencia. Es decir, se asume que la curva de riesgo de cada individuo sigue la misma curva en el tiempo. Este es un presupuesto fuerte para trabajos en ciencia política, ya que las observaciones cambian en el tiempo y se diferencian unas de otras. Piénsense en el trabajo de Altman, por ejemplo. Uno puede teorizar que la probabilidad de una iniciativa de democracia directa en un determinado año en un determinado país estará afectada por el nivel de solidez de las instituciones democráticas, que podemos medir con algún tipo de variable estándar como los 21 puntos de [Polity IV](https://www.systemicpeace.org/polity/polity4.htm) o la más reciente medición de [V-Dem](https://www.v-dem.net). Podemos, entonces, teorizar que a mayor solidez institucional mayor probabilidad de implementar mecanismos de democracia directa. 

Sin embargo, los valores de estas variables no solo difieren ente países, sino que a lo largo del tiempo estas variables cambian mucho para un mismo país. Piénsese en Colombia, por ejemplo, en que la variable de V-Dem “vdem” sufrió avances y retrocesos entre 1900 y 2016 (ver figura 3). Cada vez que el valor de esta variable cambia, necesariamente cambia la tasa de riesgo de democracia directa para Colombia, rompiendo el presupuesto de proporcionalidad de los riesgos.

<div class="figure" style="text-align: center">
<img src="00-images/surv/polyarch.png" alt=" Valores de poliarquía según V-Dem " width="80%" />
<p class="caption">(\#fig:polyarch) Valores de poliarquía según V-Dem </p>
</div>

La ventaja del modelo de Cox sobre sus contrapartes paramétricas es que existen tests para saber si alguna variable de nuestro modelo rompe el presupuesto de proporcionalidad de los riesgos, y de esa forma podremos corregirlo generando interacciones entre estas variables y variables temporales. De esta forma, permitimos que en nuestro modelo haya dos tipos de coeficientes: coeficientes constantes en el tiempo, y coeficientes cambiantes en el tiempo. Por ejemplo, podemos imaginar que, ante un aumento brusco en la calidad de las instituciones democráticas de un país, la tasa de riesgo de implementar democracia directa se dispare y que dicho efecto de desvanezca en el lapso de cuatro o cinco años. Cuando definas tu modelo, es importante que reflexiones sobre qué variables puede asumirse que permanezcan constantes en los riesgos y cuáles no. 

La recomendación dada por el *Oxford Handbook* para una buena implementación de modelos de supervivencia es la siguiente: (a) Primero, dada las ventajas de los modelos semi-paramétricos sobre los paramétricos, se recomienda el uso de Cox por sobre Weibull u otro modelo paramétrico. (b) Una vez que hemos definido nuestra variable dependiente (el evento), el tiempo de “nacimiento” y de “muerte” de cada observación, podemos especificar nuestro modelo. (c) Los coeficientes deben ser interpretados en tasas de riesgo (hazard rates), lo que exige exponenciar los coeficientes brutos que obtenemos en R.  (d) Una vez que tenemos el modelo que creemos correcto, en función de nuestras intuiciones teóricas, es necesario testear que ninguno de los coeficientes viole el presupuesto de proporcionalidad de los riesgos. Para ello ejecutamos un test de Grambsch y Therneau, o mediante el análisis de los residuos de Schoenfeld. (e) Una vez identificados los coeficientes problemáticos, permitimos que estos interactúen con el logaritmo natural de la variable que mide la duración del evento. De esta forma, permitimos que haya coeficientes cuyo efecto se desvanece o se potencia con el tiempo. Una vez corregidos los coeficientes problemáticos, podemos si, proceder a interpretar nuestro modelo y la función de supervivencia del modelo. 

## Estimación de los modelos de Cox en R

Volvamos, entonces, a la pregunta que se hizo David Altman en el capítulo 3 de [Citizenship and Contemporary Direct Democracy [-@altmanCitizenshipContemporaryDirect2019], "Catching on: waves of adoption of citizen-initiated mechanisms of direct democracy since World War I". La pregunta aquí es: ¿Por qué algunos países demoran menos que otros en implementar instancias de democracia directa? Comencemos por cargar el `tidyverse` y nuestra base (esta última, desde nuestro paquete `paqueteadp`):



```r
library(tidyverse)
library(survival)
```


```r
library(paqueteadp)
data("democracia_directa")
```

Ahora la base se ha cargado en nuestra sesión de R:


```r
ls()
## [1] "democracia_directa"
```

Las variables que tenemos en la base son las siguientes:

+ Variable dependiente, que registra la ocurrencia del evento, que en este caso es la adopción de un mecanismo de democracia directa - `dem_directa`
+ Año - `anio`  
+ Nombre del país - `pais_nombre` 
+ El país sufre un proceso de rápida democratización - `dem_rapida_positiva`
+ El país sufre un proceso rápido de deterioro de la democracia - `dem_rapida_negativa`
+ Memoria de instancias previas de democracia directa - `memoria`
+ Score de democracia del país - `vdem`
+ Efecto de la difusion de capacidades - `difusion_cap`
+ Efecto de la difusión de ocurrencias - `difusion_ocurr`
+ Logaritmo natural de la población total del país - `log_poblacion`
+ Dummy para ex colonias británicas - `colonia_gb`
+ Dummy para ex miembros de la URSS - `colonia_urss`

A lo largo del ejemplo usaremos los paquetes `skimr`, `countrycode`, `survival`, `rms`, `survminer`, `ggalt`, `tidyverse`y `texreg`, pero los iremos cargando uno a uno para que veas para que sirven. 
Si utilizamos `skim`, como ya hicimos en otros capítulos, podemos ver que es una base en formato de panel balanceado. Es decir, tenemos una variable "país" (`pais_nombre`), que se repite a lo largo de una variable "año" (`anio`).

<div class="figure" style="text-align: center">
<img src="00-images/surv/skim_surv.png" alt="Skim de nuestra base de datos" width="100%" />
<p class="caption">(\#fig:skim-surv)Skim de nuestra base de datos</p>
</div>

Los países ""entran" a la base cuando comienzan a existir como países independientes. Veamos el caso de Albania, por ejemplo, que nace como país en 1912 luego de las [Guerras los Balcanes](https://es.wikipedia.org/wiki/Guerras_de_los_Balcanes):


```r
democracia_directa %>%
  filter(pais_nombre == "Albania")
## # A tibble: 105 x 23
##   pais_nombre  anio dem_directa dem_rapida_posi~ dem_rapida_nega~ memoria
##   <chr>       <dbl>       <dbl>            <dbl>            <dbl>   <dbl>
## 1 Albania      1912           0                0                0       0
## 2 Albania      1913           0                0                0       0
## 3 Albania      1914           0                0                0       0
## # ... with 102 more rows, and 17 more variables
```

> Nota: Si tus datos están ya en este formato, puedes omitir este paso.

Para que los modelos funcionen correctamente en R, los países deberían salir del análisis (¡y de la base!) cuando "mueren". En este caso la muerte se da cuando los países adoptan mecanismos de democracia directa. Albania, siguiendo el ejemplo, debería dejar de existir en 1998, y no perdurar en la base hasta 2016 como sucede ahora. Entonces crearemos una segunda versión de nuestra base de datos donde esto ya ha sido corregido: 

<div class="figure" style="text-align: center">
<img src="00-images/surv/ex_data.png" alt="Cómo debe verse tu base de datos" width="70%" />
<p class="caption">(\#fig:surv-ex-data)Cómo debe verse tu base de datos</p>
</div>


```r
democracia_directa_b <- democracia_directa %>%
  group_by(pais_nombre) %>%
  # pedimos que la suma acumulada del dummy es como máximo 1
  filter(cumsum(dem_directa) <= 1) %>%
  ungroup()
```
Lo que estamos haciendo es filtrar los datos para que cuando ocurra eñ evento de interés (en este caso es la variable `dem_directa`) lo que viene después en el tiempo es reemplazado por NAs. Si comparamos el caso de Albania para la base original y para la base actual veremos la diferencia. En la primera base de datos tenemos observaciones para 1999 y 2000, después de que ocurriera el evento. En la segunda base, Albania "muere" en 1998 y allí se cortan los datos para el país:


```r
democracia_directa %>%
  filter(pais_nombre == "Albania" & dem_directa == 1)
## # A tibble: 19 x 23
##   pais_nombre  anio dem_directa dem_rapida_posi~ dem_rapida_nega~ memoria
##   <chr>       <dbl>       <dbl>            <dbl>            <dbl>   <dbl>
## 1 Albania      1998           1                0                0       1
## 2 Albania      1999           1                0                0       1
## 3 Albania      2000           1                0                0       1
## # ... with 16 more rows, and 17 more variables

democracia_directa_b %>%
  filter(pais_nombre == "Albania" & dem_directa == 1)
## # A tibble: 1 x 23
##   pais_nombre  anio dem_directa dem_rapida_posi~ dem_rapida_nega~ memoria
##   <chr>       <dbl>       <dbl>            <dbl>            <dbl>   <dbl>
## 1 Albania      1998           1                0                0       1
## # ... with 17 more variables
```

En resumen, ahora tenemos un panel desbalanceado, en el que los países entran a la base cuando comienzan a existir como tales y salen, o bien cuando adoptan mecanismos de democracia directa, o bien cuando la base termina en su extensión temporal (en 2016). De esta forma nuestra base se acerca mucho a la figura \@ref(fig:surv-ex-obs) con la que ejemplificamos los distintos tipos de observaciones. 

> **Ejercicio 9A.** Tómese un minuto para hacer un ejercicio antes de continuar. En el capítulo anterior utilizamos la base de datos *Democracies and Dictatorships in Latin America: Emergence, Survival, and Fall* [@mainwaringDemocraciesDictatorshipsLatin2013]. Utiliza la base de datos para ver si está lista para ser usada con modelos de supervivencia o si necesitas transformarla.

Qué tal si probamos hacer algo similar a la figura 1 pero con los datos de David Altman? Este tipo de figuras se llaman gráficos de Gantt, y pueden recrearse con `ggplot2` aunque es justo decir que hay que seguir unos cuantos pasos, y puede ser algo difícil. Ojalá que con este ejemplo puedas recrearlo con tus propios datos porque una figura así es de mucha utilidad para el lector. 

Primero debemos crear una base de datos que, para cada país, registre el año de entrada y el año de salida. También nos interesa por qué sale el país: ¿adopta democracia directa o la base termina? Vamos a crear un subonjunto que llamaremos `gantt_plot_df` donde nos quedamos solamente con tres variables de la base, que son el nombre del país `pais_nombre`, el año `anio`, y la variable dependiente `dem_directa`. También quitaremos de la base aquellas observaciones que para el primer año de la base ya han "muerto". Por ejemplo, Suiza había implementado mecanismos de democracia directa mucho antes que 1900, así que desde el primer año de la base hasta el último la variable dependiente será "1":


```r
gantt_chart_df <- democracia_directa_b %>%
  # las variables que nos interesan
  dplyr::select(pais_nombre, anio, dem_directa) %>%
  group_by(pais_nombre) %>%
  filter(anio == min(anio) | anio == max(anio)) %>%
  # necesitamos eliminar las observaciones para los países que "nacen" con 
  #democracia directa ya implementada:
  filter(!(anio == min(anio) & dem_directa == 1)) %>%
  summarize(anio_enters = min(anio),
            anio_exits  = max(anio),
            exits_bc_dd = max(dem_directa)) %>%
  ungroup()
gantt_chart_df
## # A tibble: 193 x 4
##   pais_nombre anio_enters anio_exits exits_bc_dd
##   <chr>             <dbl>      <dbl>       <dbl>
## 1 Afganistán         1919       2016           0
## 2 Albania            1912       1998           1
## 3 Andorra            1900       2016           0
## # ... with 190 more rows
```

Los países que salen debido a la democracia directa ("mueren") son:


```r
gantt_chart_df %>% filter(exits_bc_dd == 1)
## # A tibble: 48 x 4
##   pais_nombre anio_enters anio_exits exits_bc_dd
##   <chr>             <dbl>      <dbl>       <dbl>
## 1 Albania            1912       1998           1
## 2 Belarús            1991       1995           1
## 3 Belice             1981       2008           1
## # ... with 45 more rows
```

Podemos identificar en una nueva variable la región geopolítica de cada país, gracias a la función `countrycode::countrycode()` (Esto lo explicamos en detalle en el Capítulo \@ref(adv-data)). Este paquete es de gran utilidad para quienes hacen política comparada o relaciones internacionales porque facilita mucho darle códigos a los países. Lo que nos permite el paquete es asignar a cada país su región de pertenencia de manera casi automática:


```r
library(countrycode)

gantt_chart_df_region <- gantt_chart_df %>%
  mutate(region = countrycode(pais_nombre,
                              origin = "country.name", dest = "region"))
## Warning: Problem with `mutate()` input `region`.
## i Some values were not matched unambiguously: Afganistán, Arabia Saudita, Argelia, Azerbaiyán, Bahrein, Belarús, Bélgica, Belice, Bhután, Brasil, Camboya, Camerún, Canadá, Chipre, Comoras, Corea del Norte, Corea del Sur, Costa de Marfil, Croacia, Dinamarca, Egipto, Emiratos Árabes Unidos, Eslovaquia, España, Estados Unidos, Etiopía, Filipinas, Francia, Gabón, Gran Bretaña, Granada, Grecia, Haití, Hungría, Irán, Irlanda, Islandia, Islas Salomón, Italia, Japón, Kazajstán, Kirguistán, Letonia, Líbano, Libia, Lituania, Lituania 2, Malasia, Maldivas, Malí, Marruecos, Mauricio, México, Mónaco, Noruega, Nueva Zelandia, Omán, Países Bajos, Pakistán, Panamá, Papua Nueva Guinea, Perú, Polonia, República Centroafricana, República Checa, República Dominicana, Rusia, San Vicente y las Granadinas, Santa Lucía, Singapur, Siria, Suazilandia, Sudáfrica, Sudán, Sudán del Sur, Suecia, Tailandia, Taiwán, Tayikistán, Timor Oriental, Túnez, Turquía, Ucrania
## 
## i Input `region` is `countrycode(pais_nombre, origin = "country.name", dest = "region")`.
## Warning in countrycode(pais_nombre, origin = "country.name", dest = "region"): Some values were not matched unambiguously: Afganistán, Arabia Saudita, Argelia, Azerbaiyán, Bahrein, Belarús, Bélgica, Belice, Bhután, Brasil, Camboya, Camerún, Canadá, Chipre, Comoras, Corea del Norte, Corea del Sur, Costa de Marfil, Croacia, Dinamarca, Egipto, Emiratos Árabes Unidos, Eslovaquia, España, Estados Unidos, Etiopía, Filipinas, Francia, Gabón, Gran Bretaña, Granada, Grecia, Haití, Hungría, Irán, Irlanda, Islandia, Islas Salomón, Italia, Japón, Kazajstán, Kirguistán, Letonia, Líbano, Libia, Lituania, Lituania 2, Malasia, Maldivas, Malí, Marruecos, Mauricio, México, Mónaco, Noruega, Nueva Zelandia, Omán, Países Bajos, Pakistán, Panamá, Papua Nueva Guinea, Perú, Polonia, República Centroafricana, República Checa, República Dominicana, Rusia, San Vicente y las Granadinas, Santa Lucía, Singapur, Siria, Suazilandia, Sudáfrica, Sudán, Sudán del Sur, Suecia, Tailandia, Taiwán, Tayikistán, Timor Oriental, Túnez, Turquía, Ucrania
## Warning: Problem with `mutate()` input `region`.
## i Some strings were matched more than once, and therefore set to <NA> in the result: Papua Nueva Guinea,Sub-Saharan Africa,East Asia & Pacific
## 
## i Input `region` is `countrycode(pais_nombre, origin = "country.name", dest = "region")`.
## Warning in countrycode(pais_nombre, origin = "country.name", dest = "region"): Some strings were matched more than once, and therefore set to <NA> in the result: Papua Nueva Guinea,Sub-Saharan Africa,East Asia & Pacific

gantt_chart_df_region
## # A tibble: 193 x 5
##   pais_nombre anio_enters anio_exits exits_bc_dd region               
##   <chr>             <dbl>      <dbl>       <dbl> <chr>                
## 1 Afganistán         1919       2016           0 <NA>                 
## 2 Albania            1912       1998           1 Europe & Central Asia
## 3 Andorra            1900       2016           0 Europe & Central Asia
## # ... with 190 more rows
```

Como dice el warning, algunos países no fueron encontrados. Estos son los países que `countrycode` no pudo encontrar, seguramente porque los países están escritos de otra manera:


```r
gantt_chart_df_region %>%
  filter(is.na(region))
## # A tibble: 83 x 5
##   pais_nombre    anio_enters anio_exits exits_bc_dd region
##   <chr>                <dbl>      <dbl>       <dbl> <chr> 
## 1 Afganistán            1919       2016           0 <NA>  
## 2 Arabia Saudita        1946       2016           0 <NA>  
## 3 Argelia               1962       2016           0 <NA>  
## # ... with 80 more rows
```

Podemos corregir esto a mano ex-post o correr `countrycode::countrycode()` de nuevo, pero con el argumento `custom_match`:


```r
gantt_chart_df_region <- gantt_chart_df %>%
  mutate(region = countrycode(
    pais_nombre,
    origin       = "country.name",
    dest         = "region",
    custom_match = c("Korea, North" = "Eastern Asia",
                     "Kosovo" = "Eastern Europe",
                     "South Yemen" = "Western Asia",
                     "Vietnam, Democratic Republic of" = "South-Eastern
                     Asia",
                     "Vietnam, Republic of" = "South-Eastern Asia")))
## Warning: Problem with `mutate()` input `region`.
## i Some values were not matched unambiguously: Afganistán, Arabia Saudita, Argelia, Azerbaiyán, Bahrein, Belarús, Bélgica, Belice, Bhután, Brasil, Camboya, Camerún, Canadá, Chipre, Comoras, Corea del Norte, Corea del Sur, Costa de Marfil, Croacia, Dinamarca, Egipto, Emiratos Árabes Unidos, Eslovaquia, España, Estados Unidos, Etiopía, Filipinas, Francia, Gabón, Gran Bretaña, Granada, Grecia, Haití, Hungría, Irán, Irlanda, Islandia, Islas Salomón, Italia, Japón, Kazajstán, Kirguistán, Letonia, Líbano, Libia, Lituania, Lituania 2, Malasia, Maldivas, Malí, Marruecos, Mauricio, México, Mónaco, Noruega, Nueva Zelandia, Omán, Países Bajos, Pakistán, Panamá, Papua Nueva Guinea, Perú, Polonia, República Centroafricana, República Checa, República Dominicana, Rusia, San Vicente y las Granadinas, Santa Lucía, Singapur, Siria, Suazilandia, Sudáfrica, Sudán, Sudán del Sur, Suecia, Tailandia, Taiwán, Tayikistán, Timor Oriental, Túnez, Turquía, Ucrania
## 
## i Input `region` is `countrycode(...)`.
## Warning in countrycode(pais_nombre, origin = "country.name", dest = "region", : Some values were not matched unambiguously: Afganistán, Arabia Saudita, Argelia, Azerbaiyán, Bahrein, Belarús, Bélgica, Belice, Bhután, Brasil, Camboya, Camerún, Canadá, Chipre, Comoras, Corea del Norte, Corea del Sur, Costa de Marfil, Croacia, Dinamarca, Egipto, Emiratos Árabes Unidos, Eslovaquia, España, Estados Unidos, Etiopía, Filipinas, Francia, Gabón, Gran Bretaña, Granada, Grecia, Haití, Hungría, Irán, Irlanda, Islandia, Islas Salomón, Italia, Japón, Kazajstán, Kirguistán, Letonia, Líbano, Libia, Lituania, Lituania 2, Malasia, Maldivas, Malí, Marruecos, Mauricio, México, Mónaco, Noruega, Nueva Zelandia, Omán, Países Bajos, Pakistán, Panamá, Papua Nueva Guinea, Perú, Polonia, República Centroafricana, República Checa, República Dominicana, Rusia, San Vicente y las Granadinas, Santa Lucía, Singapur, Siria, Suazilandia, Sudáfrica, Sudán, Sudán del Sur, Suecia, Tailandia, Taiwán, Tayikistán, Timor Oriental, Túnez, Turquía, Ucrania
## Warning: Problem with `mutate()` input `region`.
## i Some strings were matched more than once, and therefore set to <NA> in the result: Papua Nueva Guinea,Sub-Saharan Africa,East Asia & Pacific
## 
## i Input `region` is `countrycode(...)`.
## Warning in countrycode(pais_nombre, origin = "country.name", dest = "region", : Some strings were matched more than once, and therefore set to <NA> in the result: Papua Nueva Guinea,Sub-Saharan Africa,East Asia & Pacific

gantt_chart_df_region
## # A tibble: 193 x 5
##   pais_nombre anio_enters anio_exits exits_bc_dd region               
##   <chr>             <dbl>      <dbl>       <dbl> <chr>                
## 1 Afganistán         1919       2016           0 <NA>                 
## 2 Albania            1912       1998           1 Europe & Central Asia
## 3 Andorra            1900       2016           0 Europe & Central Asia
## # ... with 190 more rows
```

Ahora no tenemos NA! Hemos logrado asignar una región a cada país de la muestra.


```r
gantt_chart_df_region %>%
  filter(is.na(region))
## # A tibble: 83 x 5
##   pais_nombre    anio_enters anio_exits exits_bc_dd region
##   <chr>                <dbl>      <dbl>       <dbl> <chr> 
## 1 Afganistán            1919       2016           0 <NA>  
## 2 Arabia Saudita        1946       2016           0 <NA>  
## 3 Argelia               1962       2016           0 <NA>  
## # ... with 80 more rows
```

Con nuestra base ya armada, podemos hacer el plot con facilidad, gracias a `ggalt::geom_dumbbell()`:


```r
library(ggalt)
gantt_chart <- ggplot(data    = gantt_chart_df_region,
                     mapping = aes(x     = anio_enters,
                                   xend  = anio_exits,
                                   y     = fct_rev(pais_nombre),
                                   color = factor(exits_bc_dd))) +
  geom_dumbbell(size_x = 2, size_xend = 2)

gantt_chart
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/unnamed-chunk-13-1.png" alt=" Gráfico de Gantt para todas las observaciones, versión simple " width="70%" />
<p class="caption">(\#fig:unnamed-chunk-13) Gráfico de Gantt para todas las observaciones, versión simple </p>
</div>

En el eje vertical tenemos los países ordenados alfabéticamente, y en el eje x hay dos variables que informan al gráfico, por un lado, el comienzo de la línea (`anio_enters`) y por otro su fin (`anio_exits`). Además, hay una tercera variable informativa que es el color de la línea, que denota si el país implementó o no una instancia de democracia directa (`exits_bc_dd`). Los países en azul son los que implementaron dicha instanca entre 1900 y 2016. 

Si bien la figura es de una enorme utilidad visual, hay que reconocer que son demasiados países para incluirla en el cuerpo de un artículo. Un enfoque posible es concentrarnos en ciertas regiones. Recordarás la función `filter` del Capítulo \@ref(data). Filtremos Sudamérica, por ejemplo:


```r
gantt_chart_sa <- ggplot(data    = gantt_chart_df_region %>% 
                          filter(region == "Latin America & Caribbean"),
                     mapping = aes(x     = anio_enters,
                                   xend  = anio_exits,
                                   y     = fct_rev(pais_nombre),
                                   color = fct_recode(factor(exits_bc_dd)))) +
  geom_dumbbell(size_x = 2, size_xend = 2)

gantt_chart_sa
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/unnamed-chunk-14-1.png" alt=" Gráfico de Gantt para Latinoamérica, versión simple" width="70%" />
<p class="caption">(\#fig:unnamed-chunk-14) Gráfico de Gantt para Latinoamérica, versión simple</p>
</div>
Podemos agregarle los años como texto para mejorar aún más la lectura de la figura:


```r
gantt_chart_sa <- gantt_chart_sa +
  geom_text(aes(label = anio_enters), vjust = -0.4) +
  geom_text(aes(x = anio_exits, label = anio_exits), vjust = -0.4)

gantt_chart_sa
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/unnamed-chunk-15-1.png" alt=" Gráfico de Gantt para Latinoamérica con los años de inicio y fin " width="70%" />
<p class="caption">(\#fig:unnamed-chunk-15) Gráfico de Gantt para Latinoamérica con los años de inicio y fin </p>
</div>
Finalmente, algunos retoques estéticos, con todo lo que hemos aprendido en el Capítulo \@ref(dataviz):


```r
library(ggplot2)
gantt_chart_sa <- gantt_chart_sa +
  labs(x = "año", y = "",
       title = " Años de entrada y salida, América del Sur ",
       color = "¿Adoptan instancias de democracia directa?") +
  theme(axis.text.x = element_blank())

gantt_chart_sa
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/unnamed-chunk-16-1.png" alt=" Versión pulida del gráfico de Gantt para Latinoamérica" width="70%" />
<p class="caption">(\#fig:unnamed-chunk-16) Versión pulida del gráfico de Gantt para Latinoamérica</p>
</div>

> **Ejercicio 9B.** Usando la base de datos de Mainwaring & Pérez-Liñán (2013) grafica un diagrama de Gantt como el anterior, mostrando las rupturas democráticas en México.

Además de gráficos de Gantt, es muy común que quien trabaja con modelos de supervivencia muestre gráficos con las curvas de supervivencia comparando dos grupos de interés. Por ejemplo, David Altman se pregunta si hubo una diferencia en el siglo XX entre países que se democratizaron rápidamente y aquellos que demoraron décadas en hacerlo respecto a la rapidez con que implementaron mecanismos de democracia directa. Este tipo de figuras no tiene valor inferencial, pero si gran valor descriptivo. Tenemos que estimar una curva de supervivencia no paramétrica, usando el método de Kaplan-Meier. 

A partir de este punto debemos hacer una modificación más a nuestra base. Seguramente tu también debas hacerlo con tus propios datos, así que presta atención. Los modelos de supervivencia no trabajan con datos de panel tradicionales, como el de nuestra base. Tenemos que convertirlos a "tiempo en riesgo". ¿Qué quiere decir esto? Quiere decir que necesitamos dos variables nuevas, una que nos diga el tiempo que la observación lleva en riesgo de morir al inicio de cada t `risk_time_at_start`y otra variable que haga lo mismo al final de cada t `risk_time_at_end`. Estas dos variables serán utilizadas a partir de ahora en los scripts del análisis de supervivencia: 


```r
democracia_directa_c <- democracia_directa_b %>%
  group_by(pais_nombre) %>%
  # eliminaremos el primer año para cada país;
  # ya que  por lógica no corre riesgo de morir ese primer año
  filter(anio != min(anio)) %>%
  mutate(risk_time_at_end   = c(1:n()),
         risk_time_at_start = c(0:(n() - 1))) %>%
  ungroup() %>%
  dplyr::select(pais_nombre, anio, risk_time_at_start, risk_time_at_end, 
                everything())
```


```r
democracia_directa_c
## # A tibble: 11,995 x 25
##   pais_nombre  anio risk_time_at_st… risk_time_at_end dem_directa
##   <chr>       <dbl>            <int>            <int>       <dbl>
## 1 Afganistán   1920                0                1           0
## 2 Afganistán   1921                1                2           0
## 3 Afganistán   1922                2                3           0
## # … with 11,992 more rows, and 20 more variables
```

Tracemos la curva de  supervivencia no paramétrica, usando el método de Kaplan-Meier:


```r
km <- survfit(Surv(time = risk_time_at_start, time2 = risk_time_at_end,
                   event = dem_directa) ~ dem_rapida_positiva,
              type      = "kaplan-meier",
              conf.type = "log",
              data      = democracia_directa_c)
```
Ahora podemos armar nuestro plot con `survminer::ggsurvplot()`


```r
library(survminer)
ggsurvplot(km, conf.int = T, 
           risk.table=T,
           legend.title = "",
           break.x.by = 20,
           legend.labs = c("Democratización rápida = 0",
                           " Democratización rápida = 1"),
           data = democracia_directa_c) +
  labs(title    = " Estimaciones de supervivencia de Kaplan-Meier ")
## Warning: Vectorized input to `element_text()` is not officially supported.
## Results may be unexpected or may change in future versions of ggplot2.
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/kaplan-1.png" alt="Kaplan-Meier Curve" width="1056" />
<p class="caption">(\#fig:kaplan)Kaplan-Meier Curve</p>
</div>

Altman tiene la hipótesis de que países que sufrieron "shocks" democratizadores fueron mucho más rápidos en implementar mecanismos directos. La figura confirma su intuición, pues vemos que la probabilidad de supervivencia de un país (léase como la probabilidad de que persista sin implementar mecanismos de democracia directa) se reduce a la mitad en los primeros cuatro años que siguen al shock democratizador. Por el contrario, países que se democratizan de a poco, no sufren este efecto. 

> **Ejercicio 9C.** Utilizando los mismos datos del ejercicio anterior: ¿Cómo se compara la curva de Kaplan-Meier entre los países que recibieron un alto apoyo político de los Estados Unidos y los que recibieron un bajo apoyo político? Para ello, utilice la variable `us_t`, que es un índice de 0 a 1, donde 1 denota un apoyo constante de los Estados Unidos en un país, y 0 denota que no se ofreció ningún apoyo de los Estados Unidos. Para comparar los dos grupos de la curva de Kaplan-Meier, cree un *dummy* que asuma 1 si el apoyo es mayor que un 0.75 y 0 en caso contrario.

## Modelos de supervivencia e interpretación sus coeficientes como Hazard Ratios.

Vamos a usar la base de Altman para estimar algunos modelos y correr los tests de Grambsch y Therneau para testear que ninguno de los coeficientes viole el presupuesto de proporcionalidad de los riesgos. No estamos replicando los modelos de su capítulo porque son algo más complejos (incluyen varias interacciones), simplemente usamos su base como referencia. 

> Nota: Aquí vale la pena hacer una aclaración respecto a replicabilidad: Si el autor utiliza Stata (por ejemplo con el comando `stcox`) van a haber algunas diferencias menores en los resultados obtenidos utilizando R. 

Primer modelo


```r
cox_m1 <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                  dem_rapida_positiva + dem_rapida_negativa + memoria + vdem,
                data   = democracia_directa_c, 
                method ="breslow")
```

Si lo desea, puede elegir expresar su modelo como una figura. Dado que estimaremos cinco modelos, usaremos una tabla en su lugar, pero es bueno para usted conocer `ggforest`


```r
ggforest(cox_m1)
## Warning in .get_data(model, data = data): The `data` argument is not
## provided. Data will be extracted from model fit.
```

<div class="figure" style="text-align: center">
<img src="09-surv_es_files/figure-html/ggforest-1.png" alt=" Representación gráfica de las tasas de riesgo del Modelo 1" width="768" />
<p class="caption">(\#fig:ggforest) Representación gráfica de las tasas de riesgo del Modelo 1</p>
</div>

Usando la función `cox.zph` hacemos el test de riesgo proporcional:


```r
test_cox_m1 <- cox.zph(cox_m1)
```

Miremos el valor global del test. Su p-valor es superior al punto de corte de `0.05`. 

Si lo desea, puede elegir graficar los resultados del test usando `ggcoxzph`. No lo haremos en los siguientes modelos, pero estas figuras pueden ir perfectamente a un apéndice en tu artículo.


```r
ggcoxzph(test_cox_m1)
```

<img src="09-surv_es_files/figure-html/unnamed-chunk-21-1.png" width="960" style="display: block; margin: auto;" />

Estimemos un segundo modelo:

```r
cox_m2 <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                  dem_rapida_positiva + dem_rapida_negativa + memoria + vdem +
                  difusion_cap,
                data   = democracia_directa_c,
                method ="breslow")
```

Y su test de riesgo proporcional:





```r
cox.zph(cox_m2)
##                      chisq df      p
## dem_rapida_positiva  1.693  1 0.1932
## dem_rapida_negativa  0.718  1 0.3968
## memoria              1.623  1 0.2026
## vdem                 6.594  1 0.0102
## difusion_cap         8.135  1 0.0043
## GLOBAL              14.825  5 0.0111
```

En este caso, el test global es apenas superior al p de corte de `0.05`, aún no se viola el supuesto de proporcionalidad. Sin embargo, hay una variable significativa en su Chi cuadrado:`difusion_cap`, con un p de `0.02`.

Ahora, un tercer modelo:

```r
cox_m3 <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                  dem_rapida_positiva + dem_rapida_negativa + memoria + vdem +
                  difusion_cap + difusion_ocurr,
                data   = democracia_directa_c,
                method ="breslow")
```

Y su test de riesgo proporcional:

```r
cox.zph(cox_m3)
##                      chisq df     p
## dem_rapida_positiva  2.092  1 0.148
## dem_rapida_negativa  0.473  1 0.491
## memoria              1.697  1 0.193
## vdem                 4.515  1 0.034
## difusion_cap         4.264  1 0.039
## difusion_ocurr       2.089  1 0.148
## GLOBAL              12.785  6 0.047
```

Aquí, el test revela un escenario similar al del primer modelo, es decir, sin problemas.

El cuarto modelo:

```r
cox_m4 <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                  dem_rapida_positiva + dem_rapida_negativa + memoria + vdem +
                  difusion_cap + difusion_ocurr + log_poblacion,
                data   = democracia_directa_c,
                method = "breslow")
```

Y su test de riesgo proporcional:

```r
cox.zph(cox_m4)
##                      chisq df     p
## dem_rapida_positiva  2.084  1 0.149
## dem_rapida_negativa  0.445  1 0.505
## memoria              1.677  1 0.195
## vdem                 4.533  1 0.033
## difusion_cap         4.224  1 0.040
## difusion_ocurr       2.059  1 0.151
## log_poblacion        2.817  1 0.093
## GLOBAL              17.867  7 0.013
```

Note como el p-valor global está muy cerca del punto de corte pero aún se mantiene sobre `0.05`. Las variables `vdem` y `log_poblacion` están violando el presupuesto de proporcionalidad de los riesgos, pues sus p-valores son menores a `0.05`. 

El quinto modelo:

```r
cox_m5 <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                  dem_rapida_positiva + dem_rapida_negativa + memoria + vdem +
                  difusion_cap + difusion_ocurr + log_poblacion + colonia_gb,
                data   = democracia_directa_c,
                method ="breslow")
```

Su test de riesgo proporcional:


```r
cox.zph(cox_m5)
##                      chisq df     p
## dem_rapida_positiva  1.883  1 0.170
## dem_rapida_negativa  0.380  1 0.538
## memoria              1.274  1 0.259
## vdem                 4.640  1 0.031
## difusion_cap         3.960  1 0.047
## difusion_ocurr       1.798  1 0.180
## log_poblacion        2.227  1 0.136
## colonia_gb           0.119  1 0.730
## GLOBAL              16.833  8 0.032
```

El quinto modelo presenta el mismo escenario que el cuarto modelo. Tenemos dos variables violando el presupuesto de proporcionalidad. El test global tiene un p-valor de `0.16` por lo que no deberíamos preocuparnos por resolver la violación. Sin embargo, caso que en tu trabajo tengas un p-valor global menor a `0.05` te mostramos como abordar el problema tal como recomienda el texto del *Oxford Handbook* sobre el que basamos la discusión teórica al inicio del capítulo. Una forma de resolverlo es interactuando las variables problemáticas con el logaritmo natural de la variable temporal que creamos anteriormente.

Si el p-valor del test global fuese menor a `0.05`, el quinto modelo corregido se vería así:


```r
cox_m5_int <- coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa) ~
                      dem_rapida_positiva + dem_rapida_negativa + memoria + vdem +
                      difusion_cap + difusion_ocurr + log_poblacion + 
                      colonia_gb + vdem:log(risk_time_at_end) +
                      log_poblacion:log(risk_time_at_end),
                    data   = democracia_directa_c,
                    method = "breslow")
## Warning in coxph(Surv(risk_time_at_start, risk_time_at_end, dem_directa)
## ~ : a variable appears on both the left and right sides of the formula
```

Verás que el test ya no muestra problemas con la proporcionalidad de los riesgos.


```r
cox.zph(cox_m5_int)
##                                        chisq df     p
## dem_rapida_positiva                 1.13e+00  1 0.287
## dem_rapida_negativa                 3.37e-01  1 0.562
## memoria                             5.39e-01  1 0.463
## vdem                                5.88e-05  1 0.994
## difusion_cap                        4.16e+00  1 0.041
## difusion_ocurr                      1.15e+00  1 0.284
## log_poblacion                       6.80e-01  1 0.410
## colonia_gb                          5.27e-02  1 0.818
## vdem:log(risk_time_at_end)          4.75e-02  1 0.828
## log_poblacion:log(risk_time_at_end) 9.49e-01  1 0.330
## GLOBAL                              1.35e+01 10 0.195
```

Veamos todos los modelos juntos con `texreg`:


```r
library(texreg)
```

Para obtener *hazard ratios* en `R` necesitamos exponenciar los coeficientes y luego calcular los errores estándar y valores-p a partir de una transformación de la matriz varianza-covarianza del modelo. En el Capítulo \@ref(logit) vimos cómo hacer esto para modelos logísticos cuando queremos *odds ratios* (utilizando las opciones `override.coef`, `override.se` y `override.pvalues` de `texreg`). Para los modelos de supervivencia este paso es idéntico. La única diferencia para nuestro caso actual es la siguiente: ahora tenemos varios modelos para los que queremos *hazard ratios*, por lo que utilizaremos la función de iteración `map()` para que las transformaciones de coeficientes, errores estándar y valores-p se apliquen en cada modelo:





```r
list_models <- list(cox_m1, cox_m2, cox_m3, cox_m4, cox_m5, cox_m5_int)

screenreg(l = list_models,
          custom.model.names = c("Modelo 1", "Modelo 2", "Modelo 3", 
                                 "Modelo 4", "Modelo 5", "Modelo 5b"),
          override.coef    = map(list_models, ~ exp(coef(.x))),
          override.se      = map(list_models, ~ odds_se(.x)),
          override.pvalues = map(list_models, ~ odds_pvalues(.x))
)
## 
## =====================================================================================================================
##                                      Modelo 1     Modelo 2      Modelo 3     Modelo 4      Modelo 5      Modelo 5b   
## ---------------------------------------------------------------------------------------------------------------------
## dem_rapida_positiva                      5.48 *       5.43 *        6.14 *       6.07 *        5.48 *        3.93 *  
##                                         (2.28)       (2.27)        (2.60)       (2.59)        (2.36)        (1.81)   
## dem_rapida_negativa                      2.26         2.37          2.35         2.36          2.22          2.30    
##                                         (1.69)       (1.78)        (1.77)       (1.78)        (1.67)        (1.74)   
## memoria                                  5.79 **      5.35 **       5.55 **      5.51 **       5.04 **       5.17 ** 
##                                         (2.07)       (1.92)        (2.00)       (1.99)        (1.84)        (1.90)   
## vdem                                     4.03         3.07          3.81         3.91          5.20        379.22    
##                                         (2.45)       (1.93)        (2.41)       (2.60)        (3.59)      (636.11)   
## difusion_cap                                          1.60 ***      2.95 **      2.94 **       2.77 **       2.70 ** 
##                                                      (0.34)        (0.93)       (0.93)        (0.88)        (0.88)   
## difusion_ocurr                                                      0.03         0.03          0.02          0.04    
##                                                                    (0.05)       (0.06)        (0.04)        (0.08)   
## log_poblacion                                                                    1.01 ***      1.01 ***      1.57 ***
##                                                                                 (0.09)        (0.10)        (0.37)   
## colonia_gb                                                                                     0.45 *        0.49 *  
##                                                                                               (0.21)        (0.24)   
## vdem:log(risk_time_at_end)                                                                                   0.26 *  
##                                                                                                             (0.12)   
## log_poblacion:log(risk_time_at_end)                                                                          0.88 ***
##                                                                                                             (0.06)   
## ---------------------------------------------------------------------------------------------------------------------
## AIC                                    377.71       374.17        370.81       372.70        371.44        365.19    
## R^2                                      0.00         0.00          0.01         0.01          0.01          0.01    
## Max. R^2                                 0.04         0.04          0.04         0.04          0.04          0.04    
## Num. events                             47           47            47           47            47            47       
## Num. obs.                            11441        11269         11260        11229         11229         11229       
## Missings                               554          726           735          766           766           766       
## PH test                                  0.05         0.01          0.05         0.01          0.03          0.19    
## =====================================================================================================================
## *** p < 0.001; ** p < 0.01; * p < 0.05
```

Agreguemos nombres a las variables para que esté lista para ser exportada:


```r
screenreg(l = list_models,
          custom.model.names = c("Modelo 1", "Modelo 2", "Modelo 3", 
                                 "Modelo 4", "Modelo 5", "Modelo 5b"),
          custom.coef.names = c("Democratización Rápida", "Rápido Retroceso Democrático", "Memoria", "Democracia", "Difusión de Capacidades", "Difusión de Ocurrencias", "Población(ln)", "Fue Colonia Británica", "Democracia x tiempo en riesgo(ln)", "Población(ln) x tiempo en riesgo(ln)"),
          override.coef    = map(list_models, ~ exp(coef(.x))),
          override.se      = map(list_models, ~ odds_se(.x)),
          override.pvalues = map(list_models, ~ odds_pvalues(.x))
)
## 
## ======================================================================================================================
##                                       Modelo 1     Modelo 2      Modelo 3     Modelo 4      Modelo 5      Modelo 5b   
## ----------------------------------------------------------------------------------------------------------------------
## Democratización Rápida                    5.48 *       5.43 *        6.14 *       6.07 *        5.48 *        3.93 *  
##                                          (2.28)       (2.27)        (2.60)       (2.59)        (2.36)        (1.81)   
## Rápido Retroceso Democrático              2.26         2.37          2.35         2.36          2.22          2.30    
##                                          (1.69)       (1.78)        (1.77)       (1.78)        (1.67)        (1.74)   
## Memoria                                   5.79 **      5.35 **       5.55 **      5.51 **       5.04 **       5.17 ** 
##                                          (2.07)       (1.92)        (2.00)       (1.99)        (1.84)        (1.90)   
## Democracia                                4.03         3.07          3.81         3.91          5.20        379.22    
##                                          (2.45)       (1.93)        (2.41)       (2.60)        (3.59)      (636.11)   
## Difusión de Capacidades                                1.60 ***      2.95 **      2.94 **       2.77 **       2.70 ** 
##                                                       (0.34)        (0.93)       (0.93)        (0.88)        (0.88)   
## Difusión de Ocurrencias                                              0.03         0.03          0.02          0.04    
##                                                                     (0.05)       (0.06)        (0.04)        (0.08)   
## Población(ln)                                                                     1.01 ***      1.01 ***      1.57 ***
##                                                                                  (0.09)        (0.10)        (0.37)   
## Fue Colonia Británica                                                                           0.45 *        0.49 *  
##                                                                                                (0.21)        (0.24)   
## Democracia x tiempo en riesgo(ln)                                                                             0.26 *  
##                                                                                                              (0.12)   
## Población(ln) x tiempo en riesgo(ln)                                                                          0.88 ***
##                                                                                                              (0.06)   
## ----------------------------------------------------------------------------------------------------------------------
## AIC                                     377.71       374.17        370.81       372.70        371.44        365.19    
## R^2                                       0.00         0.00          0.01         0.01          0.01          0.01    
## Max. R^2                                  0.04         0.04          0.04         0.04          0.04          0.04    
## Num. events                              47           47            47           47            47            47       
## Num. obs.                             11441        11269         11260        11229         11229         11229       
## Missings                                554          726           735          766           766           766       
## PH test                                   0.05         0.01          0.05         0.01          0.03          0.19    
## ======================================================================================================================
## *** p < 0.001; ** p < 0.01; * p < 0.05
```

Si bien no estamos replicando la especificación de Altman, a simple vista nuestros resultados confirmarían la hipótesis de que países que sufrieron "shocks" democratizadores fueron mucho más rápidos en implementar mecanismos directos de democracia. Hemos terminado el ejercicio, ahora te invitamos a que hagas la lista de ejercicios antes de pasar al próximo capítulo.

> **Ejercicio 9D.**
> - Utilizando `survminer`grafique una curva de Kaplan-Meier para la variable `colonia_gb`.
>
> - La variable `colonia_urss`indica aquellos países que fueron parte de la Unión Soviética. Grafique una curva de Kaplan-Meier para la variable. Incorpore esta variable a un sexto modelo, haga su test de Grambsch y Therneau y rehaga la tabla de los modelos con `texreg`
>
> - ¿Tenés tu propia base de datos de supervivencia? Sería genial que repitieras todo el ejercicio usando tus datos y compartas dudas en nuestro GitHub. Si no tienes base, usa la de Mainwaring y Pérez-Liñán (2013) para intentar identificar la variable que más aumenta los riesgos de quiebre democrático en América Latina.


