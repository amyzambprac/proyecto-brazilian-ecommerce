## proyecto-brazilian-ecommerce
# ¿Qué estados de Brasil tienen los peores tiempos de entrega?
### Rol: Junior Data Analyst

## 📊 Dashboard Interactivo

## Fase de consulta:
Olist Store es una plataforma tecnológica y de servicios de comercio electrónico de origen brasileño, fundada en 2015, que funciona como un "puente" digital. Su objetivo principal es permitir a las pequeñas y medianas empresas vender sus productos en los marketplaces más grandes (como Mercado Libre o Amazon) sin necesidad de lidiar con integraciones o logísticas complejas. 
Cuando un cliente compra un producto en Olist Store, se notifica al vendedor para que tramite el pedido. Una vez que el cliente recibe el producto, o cuando se cumple la fecha de entrega estimada, se le envía por correo electrónico una encuesta de satisfacción en la que puede valorar su experiencia de compra y escribir algunos comentarios.

El negocio desea hacer un análisis situacional logístico de los pedidos realizados desde 2016 hasta 2018 dentro del territorio brasileño:

1. Determinar el tiempo promedio de entrega (en días) por cada estado de Brasil.

2. Identifica los 5 estados con peor desempeño y los 5 con mejor desempeño logístico.

3. Analizar la variación entre el tiempo real de entrega y el tiempo estimado por la plataforma (Olist).
   
4. Analizar el rendimiento de las entregas y encontrar formas de optimizar los plazos de entrega.
   
5. Encontrar las categorías de productos que suelen generar mayor insatisfacción entre los clientes.
  
## Objetivo del análisis:

Analizar datos logísticos de pedidos E-commerce para identificar cuellos de botella en los tiempos de entrega y detectar bajos desempeños. 

## Proceso de análisis:

## 1. Preparación

### *Datos utilizados*

Para este análisis se usaron dos datasets abiertos del sitio web kaggle.com. Un solo CSV, manejable en Excel sin necesidad de combinar archivos.

### *Información de los datasets*

Se trata de una base de datos público sobre comercio electrónico en Brasil que recoge los pedidos realizados en Olist Store. El conjunto de datos contiene información sobre 100 000 pedidos realizados entre 2016 y 2018 en diversos mercados online de Brasil. Cada pedido tiene un «customer_id» único.

⚠️DISCLAIMER⚠️

1. Se trata de datos comerciales reales que han sido anonimizados, y las referencias a las empresas y socios que aparecen en el texto de la reseña se han sustituido por los nombres de las grandes casas de «Juego de Tronos».
   
2. Un pedido puede contener varios artículos.
   
3. Cada artículo puede ser suministrado por un vendedor diferente.

Imagen de referencia de un producto en el sitio web:


### *Organización de los datasets*

La base de datos contiene columnas de: estado del pedido, el precio, el pago y el envío hasta la ubicación del cliente, las características del producto y las opiniones escritas por los clientes. También contiene datos de geolocalización que relaciona los códigos postales brasileños con coordenadas de latitud y longitud.

Esquema de la base de datos:


## 2. Staging

En esta etapa se requiere entender las preguntas muy bien para poder saber que tablas necesitaré, así ahorraremos el tiempo de limpieza de datos y solo haremos ETL en lo que usaremos. Las preguntas del 1-4 requieren conocer la variable tiempo, entre tiempo de entrega, estimado de entrega, los estados y sus rendimientos logísticos, tiempo promedio de entrega, optimizar el tiempo de entrega, entonces lo que necesitamos será: tiempos de entrega y estados de Brasil. Esas variables se encuentran en el archivo llamado olist_orders_dataset y olist_customers_dataset, por supuesto que para saber esto, debimos primero hacer un diagrama E-R para saber que columnas contiene cada tabla y para eso debemos abrir cada archivo para poner saber el nombre de estas columnas y hacer el diagrama. 

Finalmente para la pregunta 5 que se trata sobre los reviews de los clientes debemos de unir varias tablas para conectar los reviews con el nombre del producto para conocer cuál producto genera insastifacción al cliente y para eso usaremos las tablas de: reviews, items, products y translation. Por lo tanto nos enfocaremos en esas únicamente, excluyendo las demás base de datos que no necesitamos. 

Luego que tenemos identificados los archivos que usaremos (en nuestro caso 6/9) los subiremos como data frame con un nombre más corto, por razones de tiempo y complejidad. Una vez que tengamos estos archivos importados y disponibles, procederemos a limpiarlos o transformarlos (como se de el caso segun el tipo de datos que tienen) y finalmente haremos consultas para encontrar las respuestas a los problemas de logística de la empresa. 

### A. *Importación*

Usamos la funcion read.csv en pandas y agregamos el tipo de archivo al final del nombre del mismo, en este caso .csv, esto para que pandas separe el texto separado por comas 

``` python
import pandas as pd

orders = pd.read_csv('olist_orders_dataset.csv')
customers = pd.read_csv('olist_customers_dataset.csv')
reviews = pd.read_csv('olist_order_reviews_dataset.csv')
products = pd.read_csv('olist_products_dataset.csv')
translation = pd.read_csv('product_category_name_translation.csv')
order_items = pd.read_csv('olist_order_items_dataset.csv')
```

Ahora ya tenemos las tablas disponibles e importadas para proceder a limpiarlas. Pero antes vamos a confirmar que están correctamente importadas viendo la estructura básica de la tabla con la funcion head. 

``` python
orders.head(3)
```


### B. *Limpieza y transformación*

Para este paso debemos saber que probablemente los datos de fechas tengan otro formato no numérico, es decir, están como *"texto"*, así que abriremos los archivos que contengan fechas relevantes para resolver las dudas, las cuales son: el tiempo/la hora en la que se realizó la compra, la de distribución del paquete y la de estimación de envío. 

Comencemos con la tabla de *orders*, buscamos dentro de su tabla contiene tres columnas importantes que son: 

1. order_purchase_timestamp
2. order_delivered_customer_date
3. order_estimated_delivery_date

### Convertir las fechas de orders

``` python
orders['order_purchase_timestamp'] = pd.to_datetime(orders['order_purchase_timestamp'])
orders['order_delivered_customer_date'] = pd.to_datetime(orders['order_delivered_customer_date'])
orders['order_estimated_delivery_date'] = pd.to_datetime(orders['order_estimated_delivery_date'])
```
Ahora queremos evaluar la logística real, entonces filtraremos los pedidos que han sido entregados, esto nos ayudará a resolver la primera pregunta (promedio en días de entrega de un pedido):

### Filtrar pedidos entregados válidos

``` python
orders_clean = orders[orders['order_status'] == 'delivered'].copy()
```

### Crear las columnas de cálculo de días

Necesitamos agregar 3 columnas en la tabla ya filtrada de pedidos 100% entregados, las cuales son:

1. real_delivery_day:
   
Restamos la fecha de envío al cliente menos la fecha de compra, para saber realmente cuántos días existen entre comprar un producto hasta que se entregó al cliente.

```python
orders_clean['real_delivery_day'] = (orders_clean['order_delivered_customer_date'] - orders_clean['order_purchase_timestamp']).dt.days
```

2. estimated_delivery_day:
   
Restamos la fecha estimada de entrega menos la fecha de compra, para saber realmente cuántos días existen entre la fecha de compra de un producto y el tiempo supuesto que estima el área logística que llegará al cliente.

```python
orders_clean['estimated_delivery_day'] = (orders_clean['order_estimated_delivery_date'] - orders_clean['order_purchase_timestamp']).dt.days
```

3. difference_days:
   
Restamos las dos columnas anteriores (estimated_delivery_day - real_delivery_day) para la diferencia entre el día que se entregó el pedido y el tiempo que se estimó en que llegara al cliente. 

```python
orders_clean['difference_days'] = (orders_clean['estimated_delivery_day'] - orders_clean['real_delivery_day'])
```

## 3. Intermediate



## Conclusiones:


## Insight:
