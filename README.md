# TC_03_SQL --- Base de datos relacional para un e-commerce tecnológico

## Descripción del proyecto

Este proyecto forma parte del Team Challenge de SQL y consiste en el
diseño e implementación de una base de datos relacional para un
**e-commerce de electrónica y accesorios tecnológicos**.

El proyecto cubre dos partes:

1.  **SQL Murder Mystery**: investigación de un caso mediante consultas
    SQL sobre una base de datos SQLite.
2.  **Modelo de datos para e-commerce**: diseño normalizado hasta 3NF,
    implementación en Google BigQuery, generación de datos sintéticos
    con Python/Faker, carga de datos y consultas analíticas.

El objetivo de la segunda parte es disponer de un modelo que permita
gestionar clientes, catálogo, pedidos, pagos y valoraciones, y analizar
ventas, productos, clientes y logística.

------------------------------------------------------------------------

# Parte 1 --- SQL Murder Mystery

La primera parte consiste en investigar un asesinato ocurrido en SQL
City mediante consultas SQL.

El notebook `investigacion.ipynb` contiene las consultas y los
resultados documentados, mientras que `investigacion_vsc.ipynb` permite
ejecutar las mismas consultas desde Python utilizando SQLite y Pandas.

La investigación utiliza 9 consultas para:

-   localizar el informe del crimen;
-   identificar al primer testigo;
-   analizar su entrevista;
-   localizar a los posibles socios del gimnasio;
-   buscar vehículos relacionados con la matrícula;
-   cruzar sospechosos con sus permisos de conducir;
-   localizar a Annabel, la segunda testigo;
-   analizar su entrevista;
-   comprobar la entrada al gimnasio.

La investigación concluye que **Jeremy Bowers es el responsable del
asesinato**, según las pistas obtenidas mediante las consultas.

------------------------------------------------------------------------

# Parte 2 --- Modelo de datos para el e-commerce

## Modelo relacional

El modelo está compuesto por siete entidades principales:

  Tabla           Descripción
  --------------- ----------------------------------
  `customers`     Clientes del e-commerce
  `categories`    Categorías de productos
  `products`      Catálogo de productos
  `orders`        Pedidos realizados
  `order_items`   Líneas de cada pedido
  `payments`      Pagos asociados a pedidos
  `reviews`       Valoraciones de líneas de pedido

### Relaciones principales

``` text
customers 1 ─── N orders
categories 1 ─── N products
orders 1 ─── N order_items
products 1 ─── N order_items
orders 1 ─── N payments
order_items 1 ─── 1 reviews
```

La relación entre `orders` y `products` es **N:M**: un pedido puede
contener varios productos y un producto puede aparecer en muchos
pedidos. Por este motivo se utiliza `order_items` como tabla intermedia.

------------------------------------------------------------------------

## Diagrama ER

El diagrama entidad-relación se encuentra en:

``` text
parte_2_modelo_bigquery/docs/er_diagram.png
```

El diagrama muestra las siete tablas, sus campos, tipos de datos, claves
primarias, claves foráneas y cardinalidades.

------------------------------------------------------------------------

# Normalización --- 3NF

El modelo se ha diseñado siguiendo las tres primeras formas normales.

## 1NF --- Primera Forma Normal

Se cumple porque:

-   cada atributo contiene un único valor;
-   no existen listas ni grupos repetidos;
-   cada columna representa un atributo atómico;
-   cada tabla dispone de una clave primaria.

## 2NF --- Segunda Forma Normal

Se cumple porque todas las tablas utilizan una clave primaria simple
(`customer_id`, `product_id`, `order_id`, etc.), por lo que no existen
dependencias parciales respecto a una clave compuesta.

Además, `order_items` separa la información propia de una línea de
pedido de la información de `orders` y `products`.

## 3NF --- Tercera Forma Normal

Se cumple porque los atributos no clave dependen directamente de la
clave primaria de su propia tabla y no de otros atributos no clave.

Por ejemplo:

-   el nombre del cliente se almacena en `customers`, no en `orders`;
-   la información de categoría se almacena en `categories`, no se
    repite en `products` como texto;
-   el nombre y el precio actual del producto se mantienen en
    `products`.

### Decisiones de diseño importantes

**`unit_price` en `order_items`**

El precio de compra se almacena en `order_items` porque representa el
precio del producto en el momento de realizar el pedido. El precio
actual de `products.sale_price` puede cambiar posteriormente.

**`country` en `customers`**

El país se mantiene directamente en `customers` porque es un atributo
del cliente y el proyecto necesita análisis geográfico. No es necesario
crear una tabla independiente de países para los requisitos actuales.

**`customer_name` no está en `orders`**

Si se almacenase el nombre del cliente junto con `customer_id`,
existiría una dependencia transitiva: el nombre dependería de
`customer_id` y no directamente de `order_id`. Por tanto, se produciría
una violación de 3NF.

------------------------------------------------------------------------

# Implementación en Google BigQuery

El notebook:

``` text
parte_2_modelo_bigquery/notebooks/01_setup_bigquery.ipynb
```

se encarga de:

1.  cargar las variables de entorno;
2.  conectarse al proyecto de Google Cloud;
3.  localizar el dataset;
4.  crear las siete tablas;
5.  definir los esquemas y tipos de datos.

Las tablas se crean respetando sus dependencias:

``` text
categories
customers
      ↓
products
      ↓
orders
      ↓
order_items
      ↓
reviews

orders
  ↓
payments
```

------------------------------------------------------------------------

# Generación de datos sintéticos

El notebook:

``` text
parte_2_modelo_bigquery/notebooks/02_generate_data.ipynb
```

utiliza:

-   Python
-   Pandas
-   Faker
-   `random`

para generar datos sintéticos realistas.

Se utiliza una semilla fija:

``` python
Faker.seed(42)
random.seed(42)
```

para favorecer la reproducibilidad.

## Volumen generado

  Tabla             Registros
  --------------- -----------
  `customers`             500
  `categories`              6
  `products`               70
  `orders`              2.000
  `order_items`         4.500
  `payments`            2.514
  `reviews`               892

Los pedidos se generan con los estados:

-   `pending`
-   `confirmed`
-   `shipped`
-   `delivered`
-   `cancelled`
-   `returned`

También se generan múltiples pagos para algunos pedidos.

Las valoraciones se generan únicamente para líneas pertenecientes a
pedidos entregados y representan aproximadamente el 35 % de las líneas
entregadas.

------------------------------------------------------------------------

# Carga de datos en BigQuery

La carga desde DataFrames se realiza mediante la función:

``` python
load_dataframe_to_bigquery(df, table_name)
```

Esta función:

1.  construye la referencia de la tabla;
2.  utiliza `load_table_from_dataframe`;
3.  espera a que finalice el proceso de carga;
4.  recupera la tabla cargada;
5.  muestra el número de filas cargadas;
6.  captura errores mediante `try/except`.

Los siete DataFrames se cargan mediante un único bucle:

``` python
dataframes = {
    "customers": customers_df,
    "categories": categories_df,
    "products": products_df,
    "orders": orders_df,
    "order_items": order_items_df,
    "payments": payments_df,
    "reviews": reviews_df
}
```

------------------------------------------------------------------------

# Queries analíticas

El notebook:

``` text
parte_2_modelo_bigquery/notebooks/03_queries_verification.ipynb
```

contiene cinco consultas analíticas:

### 1. Ingresos por mes

Calcula los ingresos mensuales a partir de pedidos y líneas de pedido.

### 2. Productos más vendidos

Obtiene los 10 productos con mayor número de unidades vendidas.

### 3. Clientes por país

Analiza la distribución de clientes por país.

### 4. Tiempo medio de entrega

Calcula el número medio de días entre la fecha del pedido y la fecha de
entrega.

### 5. Ingresos por categoría

Relaciona categorías, productos, líneas de pedido y pedidos para obtener
los ingresos por categoría.

Estas consultas permiten demostrar el funcionamiento de las relaciones
del modelo y cubrir diferentes necesidades de análisis de negocio.

------------------------------------------------------------------------


# Setup del proyecto

## 1. Clonar el repositorio

``` bash
git clone <URL_DEL_REPOSITORIO>
cd TC_03_SQL
```

## 2. Crear el entorno virtual

En Windows:

``` bash
python -m venv venv
```

Activar el entorno:

``` bash
venv\Scripts\activate
```

En macOS/Linux:

``` bash
source venv/bin/activate
```

## 3. Instalar las dependencias

``` bash
pip install -r requirements.txt
```

## 4. Configurar las credenciales de Google Cloud

Crear un archivo `.env` en la raíz del proyecto a partir de
`.env.example`.

Ejemplo:

``` env
GCP_PROJECT_ID=tu-proyecto
BQ_DATASET_ID=tu_dataset
GOOGLE_APPLICATION_CREDENTIALS=../../credentials/service-account.json
```

La cuenta de servicio debe tener permisos suficientes para acceder a
BigQuery.

### Importante

El archivo `.env` **no debe subirse a GitHub**.

Tampoco debe subirse el archivo JSON de credenciales.

El repositorio incluye:

``` text
.env.example
```

como plantilla sin credenciales reales.

------------------------------------------------------------------------

# Ejecución de los notebooks

## Parte 1

Abrir:

``` text
parte_1_sql_murder_mystery/investigacion.ipynb
```

o:

``` text
parte_1_sql_murder_mystery/investigacion_vsc.ipynb
```

La versión `investigacion_vsc.ipynb` utiliza SQLite y Pandas para
ejecutar las consultas desde Python.

## Parte 2

### Paso 1 --- Crear BigQuery

Ejecutar:

``` text
partе_2_modelo_bigquery/notebooks/01_setup_bigquery.ipynb
```

Este notebook conecta con BigQuery y crea el dataset/tablas necesarias
según la configuración de GCP.

### Paso 2 --- Generar datos

Ejecutar:

``` text
parte_2_modelo_bigquery/notebooks/02_generate_data.ipynb
```

Este notebook genera los DataFrames y permite cargarlos en BigQuery.

### Paso 3 --- Ejecutar análisis

Ejecutar:

``` text
parte_2_modelo_bigquery/notebooks/03_queries_verification.ipynb
```

Este notebook ejecuta las cinco consultas analíticas de verificación.

