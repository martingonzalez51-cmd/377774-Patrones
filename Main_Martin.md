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
