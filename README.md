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

Para la realización de esta fase he creado la clase **RealEstatePrices** dentro del paquete irm.practica.fase1. Para esta fase hay que configurar dos rutas en el objeto Utils:

- **pathFolderStreaming** que indica la ruta donde se dejaran los ficheros entrada para que se vayan leyendo en el proceso de streaming.
- **pathRealEstateCSVFileOverLimit** indica la ruta de un fichero que se copiará durante la ejecución del proceso para elevar la media de los precios. Originalmente está en la misma carpeta que el dataset de la FASE 1.

Para la funcionalidad del envío de correos hay que configurar estos parámetros, también del objeto Utils:
- **myEmail** Correo desde el que se envía el correo.
- **myEmailPass** Contraseña del correo
- **emailDest** Dirección de destino del email.

Este proceso recibe como parámetro de ejecución el límite que se comprobará que no deben pasar la media de precios de cada localidad. En el proyecto la he configurado a 7000, pero se puede cambiar en la configuración de lanzamiento del proceso. Si no se indica parámetro de ejecución por defecto se establece a 7000.

Posteriormente eliminamos de la ruta del streaming el fichero con precios que superan el límite por si existiera.

A continuación se prepara una tarea que se ejecutará 25 segundos después de iniciar la ejecución. Esta tarea copiará el fichero con precios que superan el límite en la ruta de streaming.

Creamos el objeto SparkSession y configuramos los logs para que solo muestre trazas de error. Creamos un esquema para cargar los datos de los ficheros JSON e inicializamos el DataFrame de streaming que irá leyendo los fichero que se generen. A continuación iniciamos procedimiento query y mostrar el resultado por consola de modo 'complete'.

A continuación creamos el dataframe en el que agrupamos los datos por localidad y precio medio en una ventana de una hora. Con este dataframe, filtramos las medias de precios que superen el precio límite configurado. Cada una de las filas del dataframe obtenido son las que han superado el límite. Si no hubiera ninguna fila significa que no se ha superado el límite en ningún caso. Por último iniciamos procedimiento queryLimit y realizamos un bucle por cada fila obtenida para mostrar que localidades han superado el limite y mandamos un correo de alerta al departamento correspondiente.

# Fase 3

Para la realización de esta fase 