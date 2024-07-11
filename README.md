# Análisis y Segmentación de Clientes de Productos de Lotería y Apuestas

## Problem Statement

El equipo de marketing necesita priorizar sus acciones comerciales para el año en curso. Para ello, ha solicitado al equipo de Inteligencia de Negocios un análisis detallado del comportamiento del cliente. El equipo de Inteligencia de Negocios propone implementar una segmentación de clientes utilizando la técnica RFM (Recency, Frequency, Monetary) para identificar y priorizar a los clientes de alto valor. Se acuerda también la creación de un dashboard en Power BI para visualizar los indicadores clave.

El objetivo del proyecto es desarrollar un procedimiento almacenado para generar la segmentación RFM basada en recargas realizadas por los clientes en los últimos 12 meses y crear un dashboard en Power BI que permita realizar el seguimiento de las ventas mensuales, con comparativos respecto al mes pasado y con indicadores clave del cliente.

## Descripcion de datos
Se realizará el cálculo del RFM (recency, frequency y monetary), el cual es una segmentación/agrupación de clientes que utiliza la información transaccional para brindar una etiqueta a los clientes según los criterios predefinidos.

Los datos fueron dados a traves de una base de datos dada por la universidad UPC.

- **Grupo Monetary:** de los últimos 12 meses se calcula el promedio real (solo de los meses donde ha presentado juego) y se agrupará a los clientes según el siguiente cuadro nivel. Se debe calcular sobre las recargas realizadas por los clientes.

- **Grupo Recency:** de los últimos 12 meses se calcula el último mes de recarga y se asigna un rango según la cantidad de meses que ha dejado recargar.

**Datos descriptivos**
- Producto
- Antigüedad cliente (Nuevo, 1 – 3 meses, 4 – 6 meses, 7 a 12 meses y 12 a +)
- %Recarga sobre la venta (total recarga / venta)
- Grupo Monetary
- Grupo Recency
- Descripción utilizando información relevante del cliente

**Datos numéricos**
- Venta Total
- Cantidad de clientes
- Recarga total

## Hipotesis
-  Los clientes con mas antigüedad son los que generan mas ventas y estan mas activos.
-  No existe un producto que sobresalga por sobre los demas.
-  En los ultimos meses del año el % de la recarga sobre la venta es mayor que en otros meses. 
-  Solo los clientes que realizan las recargas mas altas son fieles y generan mas ganancias.
 
## Pasos seguidos

- **Paso 1** : Conectarse a la base de datos por medio de SQL Server management Studio.
- **Paso 2** : Se genera la tabla RFM que contiene las columnas de Grupo Monetary y Grupo Recency. Ademas,cada uno con su id_cliente y codemes de ejecucion al cual pertenece.

Se ejecuta el siguiente query:

        DECLARE @MaxFecha DATE;

        SELECT @MaxFecha = MAX(FECHA)
        FROM dbo.FACT_RECARGA_CLIENTE_DIARIO;

        SELECT
                Codemes_Ejecucion,
                Id_cliente,
                CASE
                        WHEN Monetary > 50000 THEN 'Muy Alto'
                        WHEN Monetary > 10000 THEN 'Alto'
                        WHEN Monetary > 1000 THEN 'Medio'
                        WHEN Monetary > 500 THEN 'Bajo'
                        ELSE 'Muy bajo'
                END AS Grupo_Monetary,
                CASE
                        WHEN Recency < 1 THEN 'Activos'
                        WHEN Recency < 3 THEN 'Dormido 1'
                        WHEN Recency < 6 THEN 'Dormido 2'
                        WHEN Recency < 9 THEN 'Perdidos 1'
                        ELSE 'Perdidos 2'
                END AS Grupo_Recency
        FROM (
                SELECT
                        FORMAT(@MaxFecha, 'yyyy/MM') as Codemes_Ejecucion,
                        ID_CLIENTE AS Id_cliente,
                        SUM(TOTAL_RECARGA)/COUNT(DISTINCT DATEADD(MONTH, DATEDIFF(MONTH, 0, FECHA), 0)) AS Monetary,
                        DATEDIFF(MONTH, MAX(FECHA), @MaxFecha) AS Recency
                FROM [dbo].FACT_RECARGA_CLIENTE_DIARIO
                GROUP BY ID_CLIENTE
        ) AS Cliente

Resultado de la tabla RFM:

![dtink_1](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/147d990a-559b-44bb-b5b3-47caa7ffba99)


- **Paso 3** : Se importar los datos en Power BI al conectarse con la base de datos SQL Server.

- **Paso 4** : Se realizan las conexiones entre las diferentes tablas, para asi mejorar la visualizacion de los datos. Estas tablas son: FACT_RECARGA_CLIENTE_DIARIO, FACT_VENTANETA_CLIENTE_DIARIO, MAESTRO_CLIENTE, MAESTRO_MEDIO_PAGO, MAESTRO_PRODUCTO y RFM.
Y su conexion se ve de la siguiente manera:

![dtink_5](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/90a3bff1-2520-42ea-aa8b-491e81dad79a)

- **Paso 5** : Crear las siguientes 5 métricas:

  **% Recarga sobre venta:**

        %Recarga sobre venta = DIVIDE([Recarga Total], [Venta Total])

  **Cantidad de clientes:**

       Cantidad de clientes = COUNT(MAESTRO_CLIENTE[ID])

  **Recarga Total:**

        Recarga Total = SUM(FACT_RECARGA_CLIENTE_DIARIO[TOTAL_RECARGA])

  **Ultima fecha:**

        Ultima fecha = DATE(2023, 12, 31)

  **Venta Total:**

        Venta Total = SUM(FACT_VENTANETA_CLIENTE_DIARIO[TOTAL_VENTA])

- **Paso 6** : Crear la columna Antigüedad del cliente, pero para crear esta se necesita realizar la columna Meses Antigüedad del cliente, la cual es generada con el uso de la métrica última fecha.

  **Columna Meses antigüedad del cliente:**

        Meses antigüedad del cliente = DATEDIFF(MAESTRO_CLIENTE[FECHA_ALTA_SISTEMA], [Ultima fecha], MONTH)

  **Columna Antigüedad del cliente:**

        Antigüedad del cliente = SWITCH(
                TRUE(),
                MAESTRO_CLIENTE[Meses antigüedad del cliente] < 1, "Nuevo",
                MAESTRO_CLIENTE[Meses antigüedad del cliente] < 4, "1-3 meses",
                MAESTRO_CLIENTE[Meses antigüedad del cliente] < 7, "4-6 meses",
                MAESTRO_CLIENTE[Meses antigüedad del cliente] < 13, "7-12 meses",
                "12 a más"
        )

- **Paso 7** : Para aplicar un orden para las columnas Grupo_Monetary y Grupo_Recency, se creo una copia de estas columnas con el fin de aplicar el orden sin restricciones. Ademas, se crearon las dos columnas con el orden en forma numerica de estas dos.

  **Copia de Columna Grupo_Recency y Grupo_Monetary:**
        
        Grupo Monetary = RFM_Grupo2[Grupo_Monetary]
        Grupo Recency = RFM_Grupo2[Grupo_Recency]

  **Columna Monetary_value para ordenar Grupo_Monetary:**

        Monetary_Value = SWITCH(
        RFM_Grupo2[Grupo_Monetary],
        "Muy Alto", 1,
        "Alto", 2,
        "Medio", 3,
        "Bajo", 4,
        "Muy Bajo", 5
        )

  **Columna Recency_value para ordenar Grupo_Recency:**

        Recency_Value = SWITCH(
        RFM_Grupo2[Grupo_Recency],
        "Activos", 1,
        "Dormido 1", 2,
        "Dormido 2", 3,
        "Perdidos 1", 4,
        "Perdidos 2", 5
        )

Luego, a cada grupo se le aplico el orden a traves de la opcion Ordenar por columna. En donde, se ordenaron con sus respectivas columnas value.

- **Paso 8** : Para el primer Dashboard de ventas anual. Se realizan los graficos de:
  
  - **Cuadro Comparativo de Antigüedad del cliente:** Se comparan las variables de Antigüedad del cliente y Grupo Monetary. Y el valor es de la metrica Venta Total.

  - **Venta Total por Mes y Antigüedad del cliente:** Se realiza un grafico de lineas con las variables de Fecha de la tabla FACT_VENTANETA_CLIENTE_DIARIO y la metrica Venta Total. Ademas, se asigo la leyenda de Antigüedad del cliente.

  - **%Recarga sobre venta por Mes:** Se realiza un grafico de lineas con las variables de fecha de la tabla FACT_VENTANETA_CLIENTE_DIARIO y la metrica %Recarga sobre venta.

  - **Segmentación de Clientes: Recency y Antigüedad:** Se realiza un grafico de barras agrupadas horizontalmente con las variables Grupo Recency y el recuento de Id_client. Ademas, se asigno la leyenda Antigüedad del cliente.

  - **Segmentación de Clientes: Monetary y Antigüedad:** Se realiza un grafico de barras agrupadas horizontalmente con las variables Grupo Monetary y el recuento de Id_client. Ademas, se asigno la leyenda Antigüedad del cliente.

  **Nota:** La jerarquia de fechas se hizo por mes.

- **Paso 9** : Al primer Dashboard se le añaden 3 tarjetas con los valores de las metricas Cantidad de clientesn Recarga Total y Venta Total. Ademas, se añade una segmentación de datos por Producto.

- **Paso 10** : Para el segundo Dashboard de ventas de los ultimos 2 meses. Se realizan los graficos de:

  - **Venta Total del Grupo Monetary en Noviembre y Diciembre:** Se realiza un grafico de barras agrupadas horizontalmente con  las variables de fecha de la tabla FACT_VENTANETA_CLIENTE_DIARIO y la metrica Venta Total. Ademas, se asigno la leyenda Grupo Monetary.

  - **Venta Total del Grupo Recency en Noviembre y Diciembre:** Se realiza un grafico de barras agrupadas horizontalmente con  las variables de fecha de la tabla FACT_VENTANETA_CLIENTE_DIARIO y la metrica Venta Total. Ademas, se asigno la leyenda Grupo Recency.

  - **Venta Total de los productos en Noviembre y Diciembre:** Se realiza un grafico de barras agrupadas horizontalmente con  las variables de fecha de la tabla FACT_VENTANETA_CLIENTE_DIARIO y la metrica Venta Total. Ademas, se asigno la leyenda PRODUCTO de la tabla MAESTRO_PRODUCTO.

  **Nota:** La jerarquia de fechas se hizo por mes.

- **Paso 11** : Al segundo Dashboard se se añade una segmentación de datos por Fecha mes.

 
 # Report Snapshot Analisis de Ventas Anual (Power BI DESKTOP)
 
![dtink_31](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/798959dc-c0d6-4ac9-a85f-269afebad6e6)

 # Report Snapshot Analisis de Ventas ultimos 2 meses (Power BI DESKTOP)


![dtink_32](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/42a316d7-7d7c-43a3-abdf-6220baeffb94)

## Insights

A continuación, las respuestas a nuestras hipótesis y la muestra de nuevo conocimiento.

### Hipótesis 1 (Verdadero)

**Los clientes con más antigüedad son los que generan más ventas y están más activos.**

![dtink_42](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/1e9cf6d1-4d15-47de-a756-6b36454a80c4)

Según el gráfico de líneas de Venta Total por Mes y Antigüedad del Cliente, nos muestra solo tres tipos de clientes: los que tienen 12 meses a más de antigüedad, los de 7-12 meses y los de 4-6 meses. Sin embargo, se puede notar una diferencia muy notable entre los 3 tipos, siendo los de 12 meses a más los que poseen más de 150,000 en ventas totales como mínimo, siendo así mucho mayor al resto. Pues, el máximo valor de 7-12 meses es de 8,000, lo cual no alcanza ni el 10% del mínimo valor de 12 meses a más.

![dtink_44](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/2649ab86-acfd-44b6-973f-2f751cd8f72b)

Ahora, según el gráfico de barras agrupadas de Segmentación de Clientes: Recency y Antigüedad, se puede notar que la gran mayoría de los clientes activos tienen una antigüedad de entre 12 meses a más. Esto se comprueba con los siguientes datos:

Clientes Activos:
- 12 meses a más = 35
- 7-12 meses = 3
- 4-6 meses = 3
- 1-3 meses = no existe
- Nuevo = no existe

De esta manera, se puede comprobar que los clientes con más antigüedad poseen una mayor participación en las ventas.

### Hipótesis 2 (Falso)

**No existe un producto que sobresalga por sobre los demás.**


![dtink_41](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/4119c619-3df3-4ca0-b18a-df7883aa3bd6)

En el dashboard de Análisis de Ventas de los últimos 2 meses, se puede notar una distribución de los datos interesantes en el gráfico de barras agrupadas de Venta Total de los Productos en Noviembre y Diciembre, donde se encuentran dos productos que sobresalen en ventas, los cuales son Casino y Te Apuesto.

Por otro lado, si seleccionamos en el filtro y ponemos Septiembre, el cual fue el mes con más ventas del año, se puede ver que el producto con más ventas fue Casino.

Esto nos da a entender que sí existen no solo uno sino dos productos con ventas totales sobresalientes.

### Hipótesis 3 (Verdadero)

**En los últimos meses del año, el % de la recarga sobre la venta es mayor que en otros meses.**

El % de la recarga sobre la venta se explica de la siguiente manera: Si la persona recarga en su tarjeta del banco 100 y gasta 10 en algún producto, entonces el resultado del % de la recarga sobre la venta nos daría (10/100)*100, es decir, un 10%. Conociendo esto, veamos la siguiente gráfica de líneas de % Recarga sobre venta por Mes.

![dtink_43](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/dc751a8c-496b-4cbc-86dc-44f6d8af1ad3)

Se puede notar que en los últimos meses de Septiembre a Noviembre el porcentaje escaló enormemente. Y a pesar de que en Diciembre haya tenido una baja, no fue muy notable pues el porcentaje aún se mantuvo muy superior a los meses antes de Septiembre. Por lo tanto, es cierto que en los últimos meses del año el % de la recarga sobre la venta es mayor que en otros meses.

### Hipótesis 4 (Falsa)

**Solo los clientes que realizan las recargas más altas son fieles y generan más ganancias.**

Mediante el uso del gráfico de líneas de Venta Total por Mes y Antigüedad del Cliente y el cuadro comparativo de Antigüedad del Cliente, se notó que los clientes que pertenecen a las clases de Alto, Muy Alto y Medio de Monetary son los que más ventas han generado y los que en su mayoría han tenido una participación notable.

Para confirmar esto, hay que ver las siguientes estadísticas del cuadro comparativo por los meses más notables de entre Julio a Diciembre.

Julio:

![dtink_21](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/671e8cc0-adb9-4ae6-ac21-a76c8345281a)

Agosto:

![dtink_22](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/0535e8df-2d87-4d02-bc49-9b8f35db0e1b)

Septiembre:

![dtink_23](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/d4809f7e-738a-4432-9485-8de00a8bcd0f)

Octubre:

![dtink_24](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/8be32d9c-8a71-4f21-85ac-a9557ed2fade)

Noviembre:

![dtink_25](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/f23d5f31-4888-48ea-aaf3-88172bf265f4)

Diciembre:

![dtink_26](https://github.com/QuispeDiego/Analisis-Y-Segmentacion-de-Clientes-Utilizando-RFM-y-Power-BI/assets/89101133/7d18b3c4-12a6-4063-bbbe-6836053b2f5d)

A partir de estos datos, se puede resaltar el comportamiento de un cliente fiel que genera buenas ganancias, el cual no es de la clase Muy Alta de Monetary, sino que es de la clase Medio. Este tipo de cliente Medio ha sido el único que ha mantenido una participación estable, por lo que podemos considerarlo como el más fiel. Además, genera ganancias que hacen que no se sufran caídas más allá de lo previsto. Por otro lado, el cliente de clase Alta solo tuvo una participación notable entre Julio y Septiembre, al igual que la clase Muy Alta, por lo que tienen comportamientos similares, siendo lo único que los diferencia la cantidad de ventas totales que generan.

Así que, sí existe una clase aparte de la Muy Alta, que es fiel y genera ganancias.

 
 ### Perfiles de Clientes

 Las hipotesis ayudaron a generar perfiles interesantes de clientes. Este perfil de cliente se divide en dos tipos, en aquellos que son de clase monetary Medio y Muy alto. Siendo los medio los mas fieles y los que podemos considerar como el cliente estandar. Y los de Muy alto, serian como las ballenas que nos generan grandes cantidades de ganancias, pero que tienen una fidelidad menor al Medio. Ademas, se puede notar que ambos perfiles pertenecen en su mayoria a clientes de una antigüedad de 12 meses a mas, que tienen una constante participacion en general. Otra caracteristica a resaltar es que la cantidad de clientes con clase Muy alto de Monetary es la menor de todas y las de clase Medio es la Segunda mayor, siendo la clase Muy bajo la primera pero que no genera ganancias notables.

**Perfil 1: Cliente de Clase Monetaria Medio o Cliente Estandar**

- **Fidelidad**: Alta. Son los clientes más fieles.
- **Frecuencia de Compra**: Constante participación.
- **Antigüedad**: Mayoritariamente de 12 meses o más.
- **Ganancias Generadas**: Moderadas. Considerados como el cliente estándar.
- **Cantidad de Clientes**: Segunda mayor cantidad de clientes, después de la clase Muy Bajo.
- **Descripción General**: Estos clientes son esenciales para la estabilidad del negocio debido a su lealtad y participación constante. Aunque no generan las mayores ganancias individuales, su volumen y constancia los hacen muy valiosos.

**Perfil 2: Cliente de Clase Monetaria Muy Alto o Cliente Ballena**

- **Fidelidad**: Menor en comparación con los clientes de clase Medio.
- **Frecuencia de Compra**: Constante participación, pero menos frecuente que los clientes de clase Medio.
- **Antigüedad**: Mayoritariamente de 12 meses o más.
- **Ganancias Generadas**: Altas. Generan grandes cantidades de ganancias.
- **Cantidad de Clientes**: La menor cantidad de todas las clases.
- **Descripción General**: Estos clientes, conocidos como "ballenas", son muy valiosos debido a las altas ganancias que generan. Sin embargo, su menor fidelidad en comparación con los clientes de clase Medio hace que su comportamiento sea menos predecible y más volátil.

# Conclusiones finales

  Se encontro los perfiles de clientes de alto valor (Estandar y Ballena) necesarios para la generacion de nuevas estrategias,lo cual permitira un mejor manejo de las futuras ventas.

  Se pudo notar 2 productos sobresalientes (Casino y Te Apuesto) para mejorar las estrategias de venta de la empresa.

  El uso del calculo RFM, mejoro el analisis y visualizacion de los datos. Pues, es a traves de este que se pudieron generar perfiles de clientes notables.

# Recomendaciones

  Realizar comparaciones con anteriores años, para ver nuevos comportamientos y verificar resultados actuales.

  Generar estrategias basadas en recomendacion de los 2 productos notables (Casino y Te Apuesto) a los dos perfiles de clientes (Estandar y Ballena)

  Investigar nuevas caracteristicas de los clientes Ballenas para entender mejor su comportamiento atipico. Ademas, revisar si existen factores externos que generen estos comportamientos.
