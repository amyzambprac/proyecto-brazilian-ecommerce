## proyecto-brazilian-ecommerce
# ¿Qué estados de Brasil tienen los peores tiempos de entrega?
### Rol: Junior Data Analyst

## 📊 Dashboard Interactivo

## Fase de consulta:
Olist Store es una plataforma tecnológica y de servicios de comercio electrónico de origen brasileño, fundada en 2015, que funciona como un "puente" digital. Su objetivo principal es permitir a las pequeñas y medianas empresas vender sus productos en los marketplaces más grandes (como Mercado Libre o Amazon) sin necesidad de lidiar con integraciones o logísticas complejas. 

<img width="1313" height="864" alt="imagen par apdf brazilian ecommerce" src="https://github.com/user-attachments/assets/eebfc736-2b1c-474c-9979-3a2f9d277a2a" />

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

<img width="2486" height="1496" alt="FOTO ESQUEMA BRAZILIAN ECOMMERCE" src="https://github.com/user-attachments/assets/f2ec13d8-0931-4efa-95cb-22a876226e16" />

## 2. Staging

En esta etapa se requiere entender las preguntas muy bien para poder saber que tablas necesitaré, así ahorraremos el tiempo de limpieza de datos y solo haremos ETL en lo que usaremos. Las preguntas del 1-4 requieren conocer la variable tiempo, entre tiempo de entrega, estimado de entrega, los estados y sus rendimientos logísticos, tiempo promedio de entrega, optimizar el tiempo de entrega, entonces lo que necesitamos será: tiempos de entrega y estados de Brasil. Esas variables se encuentran en el archivo llamado olist_orders_dataset y olist_customers_dataset, por supuesto que para saber esto, debimos primero hacer un diagrama E-R para saber que columnas contiene cada tabla y para eso debemos abrir cada archivo para poner saber el nombre de estas columnas y hacer el diagrama. 

Finalmente para la pregunta 5 que se trata sobre los reviews de los clientes debemos de unir varias tablas para conectar los reviews con el nombre del producto para conocer cuál producto genera insastifacción al cliente y para eso usaremos las tablas de: reviews, items, products y translation. Por lo tanto nos enfocaremos en esas únicamente, excluyendo las demás base de datos que no necesitamos. 

Luego que tenemos identificados los archivos que usaremos (en nuestro caso 6/9) los subiremos como data frame con un nombre más corto, por razones de tiempo y complejidad. Una vez que tengamos estos archivos importados y disponibles, procederemos a limpiarlos o transformarlos (como se de el caso segun el tipo de datos que tienen) y finalmente haremos consultas para encontrar las respuestas a los problemas de logística de la empresa. 

## A. *Importación*

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


## B. *Limpieza y transformación*

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
Una vez ya tenemos los datos cargados, transformados y limpios, procedemos a analizar que tipo de funciones y/o operaciones debemos realizar.
Emperazaremos creando columnas de los tiempos de envío.

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

Para comprobar que las 3 columnas se crearon correctamente en una tabla con los solo los pedidos entregados (delivered), usamos:

```python
orders_clean[['real_delivery_day', 'estimated_delivery_day', 'difference_days']].head(3)
```

<img width="409" height="130" alt="image" src="https://github.com/user-attachments/assets/5f3d7683-1f61-4615-9711-f2d64529b4c3" />

## 3. Intermediate

Hacemos un merge para conocer las ubicaciones de los pedidos que si fueron entregados con la tabla de orders_clean que ya está filtrada y limpia.

```python
orders_geo = pd.merge(orders_clean, customers[['customer_id', 'customer_state']], on='customer_id', how='inner')
```

Revisamos la tabla con las columnas nuevas que se crearon

```python
orders_geo.head(3)
```

<img width="976" height="237" alt="image" src="https://github.com/user-attachments/assets/318ca7db-f370-4ca9-af58-4fbe5f9f9945" />

### Promedio del tiempo real de entrega de un paquete

Ahora sacamos la media/promedio de cada nueva columna (real_delivery_day, difference_days) para saber el promedio de días en el que un pedido llega al cliente y el promedio de la diferencia de días que existe entre el tiempo estimado y el real, esto con el objetivo de evaluar la precisión del sistema de estimación de envíos y entender qué tan bien cumple la empresa sus promesas comerciales. Este proceso corresponde a buscar la solución de la primera pregunta de este proyecto.

```python
orders_geo['real_delivery_day'].mean()
```
### Resultado:

Media = 12.093 días. El promedio de días en el que un llega un pedido a su destino es de aproximadamente 12 días.

### Promedio de la diferencia de tiempo entre el real y estimado

```python
orders_geo['difference_days'].mean()
```
### Resultado:

Media = 11.279 días. El promedio de días en el que un pedido llegue antes de los prometido es de aproximadamente 12 días.

No sacamos la media de el *'estimated_delivery_day'* ya que es solo una promesa matemática y no una realidad operativa, por lo tanto no es necesario el promedio.

Sin embargo, sacaremos la mediana de las tres columnas para confirmar la media, y con esto saber si coinciden o no se dispersa mucho de la media. Para el negocio, la mediana es lo que representa un *"cliente típico"* con respecto a tiempos de entrega de pedidos, evitando que los casos extremos o paquetes problemáticos distorsionen la realidad del negocio. 

### Mediana aritmética de los tiempos de entrega

```python
orders_geo[['real_delivery_day', 'estimated_delivery_day', 'difference_days']].describe()
```
### Resultado:

En la imagen adjunta observamos que la mediana no varía de la media, eso indica que los valores se encuentran dentro de lo normativo y no hay datos erronéos alterándolos. 

<img width="432" height="271" alt="image" src="https://github.com/user-attachments/assets/77f7ff0c-4afe-4435-a795-4148866bae28" />


### Top 5 estados con los envíos más lentos (en promedio)

Para encontrar la respuesta de la pregunta 2 (Identifica los 5 estados con peor desempeño y los 5 con mejor desempeño logístico) usaremos la función de agrupación (groupby), media (mean) y orden de mayor a menor sobre la tabla limpia que tenemos (orders_geo). Usamos la función de promedio para tener una vista general de los tiempos de entrega. Agrupamos por estado y ordenamos para tener un ranking de los estados.

```python
orders_geo.groupby('customer_state')['real_delivery_day'].mean().sort_values(ascending=False).head(5)
```

### Resultado:

El estado Roraima (RR) tiene el PEOR desempeño logístico con un promedio de 28.97 días de entrega de un pedido, aproximadamente 4 días después del tiempo estimado por la empresa (23.37 días)

<img width="301" height="127" alt="image" src="https://github.com/user-attachments/assets/97ac62bc-8c6a-400c-9f24-25dd753eeea2" />


### Top 5 estados con los envíos más rápidos (en promedio)

```python
orders_geo.groupby('customer_state')['real_delivery_day'].mean().sort_values(ascending=True).head(5)
```

### Resultado:

El estado São Paulo (SP) tiene el MEJOR desempeño logístico con un promedio de 8.29 días de entrega de un pedido, aproximadamente 16 días antes de la fecha prevista por la emprea (23.37días) y 4 días antes de la diferencia entre el real y el estimado (11.27 días)

<img width="304" height="123" alt="image" src="https://github.com/user-attachments/assets/cd1f5912-6607-40c2-b00e-9381d7c1a340" />


Ahora que tenemos los tiempos promedios y los estados de brasil unidos mediante un join, podemos resolver la pregunta 1.

### Tiempo promedio de entrega por cada estado

```python
promedio_por_estado = orders_geo.groupby('customer_state')['real_delivery_day'].mean().round(1).sort_values()
```
La comprobación del resultado:

```python
promedio_por_estado.head(5)
```

<img width="306" height="168" alt="image" src="https://github.com/user-attachments/assets/a2b96f79-5aec-4b03-a429-61f1cf7a62fc" />

### Resultado:

Sao Paulo es el estado con el mejor promedio de días de entrega, con un resultado de 8.3 días, comprobando lo que anteriormente ya habíamos calculado en el raking de los estados con los tiempos de entrega más rápidos; seguido de Minas Gerais y Paraná con un tiempo de entrega igual de 11.5 días.


Para resolver la pregunta 5 (encontrar las categorías de productos que suelen generar mayor insatisfacción entre los clientes) debemos hacer un merge/join entre los reviews de los clientes, los artículos o items que tienen un reviews y el # de orden o transacción. 

### Categorías de productos que tienen mayor insatisfacción entre los clientes 

```python
df_reviews = order_items.merge(products, on='product_id').merge(reviews, on='order_id')
```
Luego hacemos una agrupación de las categorías de los productos, usamos las funciones count y mean para calcular cuántos productos hay por categoría y el promedio de las puntaciones de las reseñar/reviews.

```python
cate_summary = df_reviews.groupby('product_category_name')['review_score'].agg(['mean', 'count'])
```

Y ahora buscamos calcular las categorías que generan insastifacción según las reviews de los clientes, también ajustamos con un filtro de > 50 para que solo nos aparezcan las categorías que alcanzaron más de 50 puntos en las puntuaciones de los reviews de los clientes para descartar que se filtren categorías con pocas ventas y mala puntuación que llegarán a distorsionar la tabla resultante. Ordenamos para que nos aparezcan los productos con más insastifacción primero y comprobamos el resultado con solo 10 filas.

```python
cate_insatisfaccion = cate_summary[cate_summary['count'] > 50].sort_values(by='mean', ascending=True).head(10)
```
<img width="348" height="398" alt="image" src="https://github.com/user-attachments/assets/d22c0ccb-b6ee-4ad3-b8c5-6c0093d54d29" />

Ahora tenemos el promedio de las reviews según cada categoría y la cantidad de productos que se vendieron de las mismas.

### Resultado:

La categoría con mayor insastifacción evaluada por los clientes lleva por nombre 'Moveis escritório' o muebles de oficina con un promedio de insastifaccion de 3.49/5 y 1687 artículos vendidos. 

Finalmente para responder la pregunta 4 lo definiremos en la sección de conclusiones e insights.


## Marts 

Se definieron e implementaron dos tablas estructuradas (modelado dimensional) para separar las métricas cuantitativas del análisis logístico de los atributos geográficos.

### 1. Tabla de Hechos Logística

Almacena los indicadores numéricos y fechas relativas a cada orden de compra.

```python
fact_orders = orders_clean[['order_id', 'customer_id', 'order_status', 'real_delivery_day', 'estimated_delivery_day', 'difference_days']].copy()
```

### 2. Dimensión de Clientes/Geografía

```python
dim_customers = customers[['customer_id', 'customer_unique_id', 'customer_zip_code_prefix', 'customer_city', 'customer_state']].copy()
```

## Conclusiones:

1. Se encontró el rendimiento actual de las entregas en la cual existe una alta diferencia de días entre los tiempos de entregas dependiendo el estado en donde el cliente viva, siendo esta diferencia de 20 días, dando como resultado una alta brecha geográfica entre comsumidores. Por ejemplo, un cliente que recibe un pedido en Sao Paulo tiene recibe su pedido en un aproximado de 8 días mientras que alguien que recibe su pedido en el estado de Roraima tiene que esperar 29 días por el mismo. 
   
2. La mayoría de los vendedores y almacenes están agrupados en la región Sureste (SP, PR, MG). Cada pedido enviado al Norte o Nordeste debe recorrer distancias de más de 2,000 a 3,000 km con una infraestructura vial posiblemente compleja.

3. Existe un aceptación positiva al recibir el pedido porque la empresa promete entregas en 23.4 días promedio cuando el tiempo real de entrega global es de 12.1 días y esto genera sorpresas positivas (+11.3 días de anticipación) en los consumidores. Sin embargo, si en el checkout se menciona esto pordría generar que potenciales clientes abandonen la compra por considerar que el envío es "demasiado lento o tardío, o no llegará para cuando lo necesito".


## Insight:
