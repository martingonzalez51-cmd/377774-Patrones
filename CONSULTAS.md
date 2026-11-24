# &nbsp;Consultas SQL y Obtención de Hallazgos

**Proyecto:** Diagnóstico Territorial y Económico para Estrategia de Desarrollo Comercial en Campaña Política  
**Municipio analizado:** Ensenada, Baja California (`cve_ent = 02`, `cve_mun = 001`)  
**Motor de base de datos:** PostgreSQL

## Fase 1: Carga, Limpieza y Preparación de Datos (Para Puntos 4.3 y 7.1)

### 1.1 Creación de tabla receptora e importación

* **¿Qué hace?** Crea una tabla temporal con las 42 columnas oficiales del DENUE en formato `TEXT` para evitar errores de tipo de dato durante la importación masiva del archivo CSV de Baja California.

```sql
CREATE TABLE denue_bc_raw (
    id TEXT, clee TEXT, nom_estab TEXT, raz_social TEXT,
    codigo_act TEXT, nombre_act TEXT, per_ocu TEXT,
    tipo_vial TEXT, nom_vial TEXT, tipo_v_e_1 TEXT, nom_v_e_1 TEXT,
    tipo_v_e_2 TEXT, nom_v_e_2 TEXT, tipo_v_e_3 TEXT, nom_v_e_3 TEXT,
    numero_ext TEXT, letra_ext TEXT, edificio TEXT, edificio_e TEXT,
    numero_int TEXT, letra_int TEXT, tipo_asent TEXT, nomb_asent TEXT,
    tipocencom TEXT, nom_cencom TEXT, num_local TEXT, cod_postal TEXT,
    cve_ent TEXT, entidad TEXT, cve_mun TEXT, municipio TEXT,
    cve_loc TEXT, localidad TEXT, ageb TEXT, manzana TEXT,
    telefono TEXT, correoelec TEXT, www TEXT, tipounieco TEXT,
    latitud TEXT, longitud TEXT, fecha_alta TEXT
);

-- Ejecutar en psql (o usar Import/Export en pgAdmin con codificación LATIN1):
-- \copy denue_bc_raw FROM 'C:/ruta/denue_inegi_02_.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',', ENCODING 'LATIN1');
```

### 1.2 Filtrado de Ensenada y auditoría de calidad de datos

* **¿Qué hace?** Extrae únicamente los establecimientos pertenecientes al municipio de Ensenada (`cve_mun = 1`), limpia espacios en blanco, estandariza en mayúsculas los nombres de colonias (etiquetando los vacíos como `'SIN COLONIA'`) y convierte las coordenadas a formato numérico.  
* **¿Para qué sirve en el informe?** El conteo total te da el dato exacto para el **Punto 4.3 (Número de registros utilizados)**, y el conteo de registros `'SIN COLONIA'` sirve como evidencia directa para el **Punto 7.1 (Calidad y confiabilidad de los datos)**.

```sql
CREATE TABLE denue_ensenada AS
SELECT 
    id AS id_establecimiento,
    TRIM(nom_estab) AS nom_estab,
    TRIM(codigo_act) AS codigo_scian,
    TRIM(nombre_act) AS nombre_act,
    TRIM(per_ocu) AS per_ocu,
    UPPER(TRIM(COALESCE(NULLIF(nomb_asent, ''), 'SIN COLONIA'))) AS colonia,
    TRIM(ageb) AS ageb,
    NULLIF(latitud, '')::DOUBLE PRECISION AS latitud,
    NULLIF(longitud, '')::DOUBLE PRECISION AS longitud,
    TRIM(fecha_alta) AS fecha_alta
FROM denue_bc_raw
WHERE cve_mun::INT = 1;

-- Resultado 1: Total de registros para el Punto 4.3
SELECT COUNT(*) AS total_registros_ensenada 
FROM denue_ensenada;

-- Resultado 2: Evidencia de inconsistencias de colonia para el Punto 7.1
SELECT 
    COUNT(*) FILTER (WHERE colonia = 'SIN COLONIA') AS registros_sin_colonia,
    ROUND(100.0 * COUNT(*) FILTER (WHERE colonia = 'SIN COLONIA') / COUNT(*), 2) AS pct_sin_colonia
FROM denue_ensenada;
```

&nbsp;

## Fase 2: Consultas Analíticas para los Hallazgos (Para Puntos 6 y 11.1)

### Hallazgo 1: Predominio de negocios de supervivencia y ausencia de empresas ancla

* **Descripción del hallazgo:** Demuestra que las colonias rezagadas sostienen su economía mediante microcomercios familiares de autoempleo (0 a 5 personas), careciendo de empresas medianas o grandes (supermercados, cadenas, maquiladoras) que inyecten salarios formales.  
* **¿Qué hace la consulta?** Agrupa los establecimientos por colonia, contabiliza cuántos pertenecen al estrato de supervivencia (`0 a 5 personas`) frente a empresas ancla (`51 personas en adelante`) y ordena aquellas colonias con actividad comercial pero cero empresas grandes.

```sql
SELECT 
    colonia,
    COUNT(*) AS total_negocios,
    COUNT(*) FILTER (WHERE per_ocu = '0 a 5 personas') AS micro_supervivencia,
    COUNT(*) FILTER (WHERE per_ocu IN ('51 a 100 personas', '101 a 250 personas', '251 y más personas')) AS empresas_ancla,
    ROUND(100.0 * COUNT(*) FILTER (WHERE per_ocu = '0 a 5 personas') / COUNT(*), 2) AS pct_supervivencia
FROM denue_ensenada
WHERE colonia <> 'SIN COLONIA'
GROUP BY colonia
HAVING COUNT(*) >= 15
ORDER BY empresas_ancla ASC, total_negocios DESC
LIMIT 15;
```

&nbsp;

### Hallazgo 2: Desconexión financiera y logística en colonias pobladas

* **Descripción del hallazgo:** Evidencia la falta de acceso a servicios bancarios y centros de distribución cercanos, lo cual encarece el surtido de mercancía e impide el acceso a financiamiento formal para los comerciantes locales.  
* **¿Qué hace la consulta?** Filtra los establecimientos financieros (código SCIAN `5221`: banca múltiple y cajeros) y logísticos (`4931`: almacenamiento y bodegas), calcula el centroide geográfico de cada colonia y mide mediante la fórmula de Haversine la distancia mínima en kilómetros hacia el banco o bodega más cercano, filtrando aquellas que quedan fuera de un radio de 3 km.

```sql
WITH infraestructura AS (
    SELECT latitud, longitud
    FROM denue_ensenada
    WHERE LEFT(codigo_scian, 4) IN ('5221', '4931')
      AND latitud IS NOT NULL
),
centroide_colonias AS (
    SELECT 
        colonia,
        COUNT(*) AS total_negocios,
        AVG(latitud) AS lat_col,
        AVG(longitud) AS lon_col
    FROM denue_ensenada
    WHERE colonia <> 'SIN COLONIA' AND latitud IS NOT NULL
    GROUP BY colonia
)
SELECT 
    c.colonia,
    c.total_negocios,
    ROUND(MIN(
        6371 * ACOS(
            LEAST(1.0, COS(RADIANS(c.lat_col)) * COS(RADIANS(i.latitud)) * COS(RADIANS(i.longitud) - RADIANS(c.lon_col)) + 
            SIN(RADIANS(c.lat_col)) * SIN(RADIANS(i.latitud)))
        )
    )::NUMERIC, 2) AS km_a_banco_o_bodega_mas_cercano
FROM centroide_colonias c
CROSS JOIN infraestructura i
GROUP BY c.colonia, c.total_negocios
HAVING MIN(
        6371 * ACOS(
            LEAST(1.0, COS(RADIANS(c.lat_col)) * COS(RADIANS(i.latitud)) * COS(RADIANS(i.longitud) - RADIANS(c.lon_col)) + 
            SIN(RADIANS(c.lat_col)) * SIN(RADIANS(i.latitud)))
        )
    ) > 3.0
ORDER BY km_a_banco_o_bodega_mas_cercano DESC;
```

&nbsp;

### Hallazgo 3: Formación de "zonas frías" (clústeres de colonias sin comercio)

* **Descripción del hallazgo:** Identifica sectores con escasez crítica de tiendas y servicios básicos, obligando a sus habitantes a desplazarse a otras zonas del municipio y fugando el gasto fuera de su comunidad.  
* **¿Qué hace la consulta?** Lista las colonias registradas en Ensenada que cuentan con 5 o menos establecimientos activos en total, permitiendo ubicar los polígonos con menor densidad económica para priorizar incentivos territoriales.

```sql
SELECT 
    colonia,
    COUNT(*) AS total_establecimientos
FROM denue_ensenada
WHERE colonia <> 'SIN COLONIA'
GROUP BY colonia
HAVING COUNT(*) <= 5
ORDER BY total_establecimientos ASC, colonia ASC;
```

&nbsp;

### Hallazgo 4: Alta concentración en un solo giro y falta de diversificación

* **Descripción del hallazgo:** Muestra que en diversas colonias no existe variedad comercial; hasta el 70% o más de los locales pertenecen exactamente al mismo giro (como tiendas de abarrotes), generando competencia directa por los mismos clientes y carencia de otros servicios.  
* **¿Qué hace la consulta?** Utiliza una función de ventana (`OVER PARTITION BY`) para calcular el peso porcentual de cada actividad económica respecto al total de negocios de su colonia, filtrando los casos donde un solo giro alcanza o supera el 70% de concentración.

```sql
WITH conteo_giros AS (
    SELECT 
        colonia,
        codigo_scian,
        nombre_act,
        COUNT(*) AS negocios_giro,
        SUM(COUNT(*)) OVER (PARTITION BY colonia) AS total_colonia
    FROM denue_ensenada
    WHERE colonia <> 'SIN COLONIA'
    GROUP BY colonia, codigo_scian, nombre_act
)
SELECT 
    colonia,
    nombre_act AS giro_dominante,
    negocios_giro,
    total_colonia,
    ROUND(100.0 * negocios_giro / total_colonia, 2) AS pct_concentracion
FROM conteo_giros
WHERE total_colonia >= 5
  AND (100.0 * negocios_giro / total_colonia) >= 70.0
ORDER BY pct_concentracion DESC, total_colonia DESC;
```

&nbsp;

### Hallazgo 5: Deterioro del centro comercial tradicional frente a expansión periférica

* **Descripción del hallazgo:** Contrasta el estancamiento en la apertura de nuevos comercios en el centro tradicional frente al crecimiento espontáneo y sin planeación en las zonas periféricas del municipio.  
* **¿Qué hace la consulta?** Compara el volumen histórico de establecimientos frente a las incorporaciones recientes (`fecha_alta` de 2025 en adelante), calculando la tasa de dinamismo comercial entre la Zona Centro y el resto de las colonias periféricas.

```sql
SELECT 
    CASE 
        WHEN colonia LIKE '%CENTRO%' THEN 'Centro Tradicional'
        ELSE 'Periferia y Otras Colonias'
    END AS zona,
    COUNT(*) AS total_historico,
    COUNT(*) FILTER (WHERE fecha_alta >= '2025-01') AS altas_recientes,
    ROUND(100.0 * COUNT(*) FILTER (WHERE fecha_alta >= '2025-01') / COUNT(*), 2) AS tasa_apertura_reciente
FROM denue_ensenada
WHERE colonia <> 'SIN COLONIA'
GROUP BY 1;
```

&nbsp;