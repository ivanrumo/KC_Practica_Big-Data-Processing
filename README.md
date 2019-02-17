# KC_Practica_Big-Data-Processing

La práctica la he realizado en un proyecto de IntelliJ. El proyecto se encuentra en el fichero practica-mod-bd-processing.zip. Los fuentes se encuentran dentro del paquete irm.practica. Por cada fase de la práctica he creado un paquete que he denominado fase1, fase2 y fase3. Además he creado el paquete utils con donde incluiré las clases y objetos de propósito general. Dentro de este paquete está el object Utils que contiene métodos de propósito general y parámetros para configurar las rutas de entrada y salida de los procesos.

# Fase 1

Para la realización de esta fase he creado la clase **RealEstatePrices** dentro del paquete irm.practica.fase1. Para esta fase hay que configurar dos rutas en el objeto Utils:

* **pathRealEstateCSVFile** que indica la ruta del fichero de entrada con el dataset con los precios de productos inmobiliarios.
* **pathRealEstateAvgPricesByLocation** indica la ruta donde se va a generar el fichero JSON con los datos resultado de esta fase

Para empezar creamos el objeto SparkSession y configuramos los logs para que solo muestre trazas de error.
A continuación cargamos el fichero csv en un dataframe con el método spark.read.csv indicando como parámetros que el fichero usa el separador ",", que el fichero tiene cabecera y que infiera el esquema.

El siguiente paso es hacer un poco de limpieza y pasar los campos de moneda de dolares a euros y los campos de tamaño de pies a metros. Para realizar esta limpieza y cambios en los datos voy a usar _User definition functions_.
La limpieza la realizamos en el campo Location ya que en algunas filas hay espacios al principio que nos dificultaran sacar datos correctos posteriormente. Así que he creado un udf que haga un trim de los valores de este campo
Para pasar el campo Size a metros he creado una udf que multiplique el valor de este campo por 10,764 que proporción de pies a metros.
Para Price a euros he creado una función en Utils que se llama getExchangeValue() que invoca al API REST de la página **exchangeratesapi.io** que nos proporciona el valor del cambio Eur-Usd publicado por el Banco Central Europeo. Así que el valor del campo Price se multiplica por el valor de cambio que hemos obtenido del API.

Una vez tenemos las udf, las usamos invocando al método select del dataframe del csv. Así obtenemos un dataframe con el campo Location limpio, el campo Price en Euros y el campo Size en metros. 

Ahora ya podemos obtener los datos deseados de esta fase. Para ello tenemos dos opciones. Yo incluyo las dos en el código, aunque una de ellas la dejo comentada:

1. Podemos usar los métodos de SparkSQL
2. Podemos crear una tabla temporal del dataframe y realizar una consulta SQL

Con ambas opciones obtenemos el mismo resultado. Yo he dejado en el código el uso de SparkSQL y dejado comentado el uso de la consulta SQL.

Por último llamamos al método coalesce para intentar reducir el número de particiones a 1 y guardamos el fichero.

# Fase 2

Para la realización de esta fase he creado la clase **ReaEstateStreaming** dentro del paquete irm.practica.fase2. Para esta fase hay que configurar una rutas en el objeto Utils:

- **pathRealEstatePricesChekpointStreaming** que indica la ruta donde se dejaran los ficheros de control del proceso de streaming.

Para la funcionalidad del envío de correos hay que configurar estos parámetros, también del objeto Utils:
- **myEmail** Correo desde el que se envía el correo.
- **myEmailPass** Contraseña del correo
- **emailDest** Dirección de destino del email.

Este proceso recibe como parámetro de ejecución el límite que se comprobará que no deben pasar la media de precios de cada localidad. En el proyecto la he configurado a 7000, pero se puede cambiar en la configuración de lanzamiento del proceso. Si no se indica parámetro de ejecución por defecto se establece a 7000.

Posteriormente eliminamos de la ruta del streaming el fichero con precios que superan el límite por si existiera.

Creamos el objeto SparkSession y configuramos los logs para que solo muestre trazas de error. Creamos un esquema para cargar los datos de los ficheros JSON e inicializamos el DataFrame de streaming que irá leyendo los fichero que se generen. 

A continuación creamos el dataframe en el que agrupamos los datos por localidad y precio medio en una ventana de una hora. A continuación iniciamos procedimiento query y mostrar el resultado por consola de modo 'complete'.

Con este dataframe, filtramos las medias de precios que superen el precio límite configurado. Cada una de las filas del dataframe obtenido son las que han superado el límite. Si no hubiera ninguna fila significa que no se ha superado el límite en ningún caso. Por último iniciamos procedimiento queryLimit y realizamos un bucle por cada fila obtenida para mostrar que localidades han superado el limite y mandamos un correo de alerta al departamento correspondiente.

Antes de dejar los procedimientos preparamos una tarea que se ejecutará 15 segundos después de iniciar la ejecución. Esta tarea realizará un proceso similar al de la fase 1, se leerá el mismo dataset, se hará limpieza de datos y se transformarán los dolares a euros y los pies a metros, pero después cogemos 5 filas aleatorias y multiplicamos el precio por 5 y dividimos el tamaño por 2. De esta manera es bastante seguro que superemos el valor medio límite configurados. El resultado se guarda en el directorio de streaming, por lo que debería de salir avisos por consola indicando que se ha superado el límite y mandar los correspondientes correos.


# Fase 3

Para la realización de esta fase he creado la clase **RealEstateML** dentro del paquete irm.practica.fase3. Para esta fase hay que configurar dos rutas en el objeto Utils:

- **pathRealEstateML_LR** que indica la ruta donde se se guardaran los resultados de la predicción con el algoritmo de Linear Regression.
- **pathRealEstateML_RF** que indica la ruta donde se se guardaran los resultados de la predicción con el algoritmo de Random Forest Regressor.

Para la realización de de esta fase he utilizado dos algoritmos de machine learning de los múltiples que provee Spark: Linear Regression y Random Forest Regressor. Linear Regression lo he usado porque lo vimos en clase y Random Forest Regressor no lo he usado por ninguna razón especial. Ambos algoritmos, en la mayoría de casos, se desvían bastante en la predicción del precio real, aunque en algunos casos se acercan bastante. Si que he podido ver que Random Forest Regressor aproxima las predicciones mejor que Linear Regression, aunque, como he dicho, no son muy fiables. Entiendo que se necesitan muchas mas muestras para que los resultados sean más certeros.

Para empezar creamos el objeto SparkSession y configuramos los logs para que solo muestre trazas de error.
A continuación cargamos el fichero csv en un dataframe con el método spark.read.csv indicando como parámetros que el fichero usa el separador ",", que el fichero tiene cabecera y que infiera el esquema.

Como nos pasa en la fase 1, la localidad tiene espacios en los nombres que nos sobran, por lo que tenemos que hacer un poco de limpieza. Una vez limpiado creamos los conjuntos de datos de entrenamiento (80%) y de pruebas (20%).

Unos de los campos con los que vamos a entrenar nuestros modelos es la Localidad. Como es un dato de tipo texto, tenemos que transformarlo en un dato numérico. Para usamos un objeto StringIndexer que asociará cada nombre de localidad a un índice.

Lo siguiente es crear un objeto VectorAssembler con las columnas que vamos a usar como features. Las columnas que vamos a indicar como features son la Localidad (en realidad usamos los datos generados con el StringIndexer), Size, Bedrooms y Bathrooms. En un principio probé con la localidad y el precio. Posteriormente probé a incluir las habitaciones y los baños y las predicciones se acercaban más a la realidad, por lo que parece que estas columan afectan al precio además del tamaño y la situación.

Después se crea y se configura el objeto LinearRegression del modelo indicando la columna Label (el precio) y las columnas features.

Para realizar el entrenamiento vamos a usar un Pipeline en el que definimos como fases el objeto StringIndexer, el VectorAssembler y el objeto LinearRegression. Con el Pipeline creado lo entrenamos con el conjunto de datos de entrenamiento. Luego probamos el modelo con el conjunto de datos de test y de los datos obtenidos obtenemos un dataframe con las columnas features, label y la predicción del modelo. El contenido del dataframe lo guardamos en la ruta configurada en el parámetro de configuración pathRealEstateML_LR.

Ahora toca realizar el modelo y la predición con el algoritmo Random Forest Regressor. Los pasos son básicamente los mismos. El mayor cambio es que obtenemos un objeto RandomForestRegressor para entrenar el modelo en vez de un objeto LinearRegression. Por el resto es igual: obtenemos un Pipiline con las mismas fases, entrenamos el modelo con los datos de entrenamiento y lo probamos con los datos de test. Los resultados del entrenamiento los guardamos en la ruta configurada en el parámetro pathRealEstateML_RF.