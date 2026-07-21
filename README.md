# Emergencias de Salud Pública en Ecuador (2023-2024)

Proyecto de fin de semestre de Análisis de Datos (ESFOT - EPN, 2026-A). Analiza la incidencia de emergencias atendidas en salud pública, cruzando datos del MSP, egresos hospitalarios del INEC, el catálogo CIE-10 y el clima por provincia, para identificar patrones de demanda y su posible relación con condiciones climáticas.

Stack: **KNIME** (ETL), **Python** (EDA y detección de anomalías), **Power BI** (dashboard), sobre una arquitectura mixta de bases SQL y NoSQL.

## Equipo

- Marjorie Valdivieso
- José Buñay 

## Estructura del repositorio

```
├── knime/
│   └── ProyectoFinal.knwf        # workflow del ETL (6 metanodos)
├── python/
│   └── eda_salud_ecuador.ipynb   # EDA, validación de calidad y Isolation Forest
├── powerbi/
│   └── Proyecto_Analisis_de_Dato.pbix            # modelo de datos y dashboard
├── docs/
│   └── Informe_IEEE.docx         # informe técnico completo (formato IEEE)
└── README.md
```

## Fuentes de datos

| Fuente | Descripción | Formato |
|---|---|---|
| MSP Emergencias (2023-2024) | Atenciones de emergencia por establecimiento de salud | CSV |
| INEC Egresos Hospitalarios (2023-2024) | Egresos con fecha de ingreso/egreso y causa CIE-10 | CSV |
| Catálogo CIE-10 Ecuador | Enfermedades y metadatos | CSV → NoSQL |
| Clima diario por provincia | Temperatura y lluvia | JSON → NoSQL |

En total, aproximadamente 4 millones de filas entre las dos fuentes principales.

## Arquitectura

1. **Staging**: los datos ya limpiados en KNIME se cargan en MySQL (fuentes estructuradas: emergencias, egresos) y en MongoDB (fuentes semiestructuradas: clima, catálogo CIE-10).
2. **Caché**: el catálogo CIE-10 se replica en Redis como hashes (`cie10:{codigo}`) para acelerar las búsquedas de diagnóstico. Se carga mediante un nodo Python embebido dentro del propio workflow de KNIME.
3. **Integración**: las 4 fuentes se unen por fecha, provincia y código CIE-10, y se calculan los campos derivados.
4. **Data Warehouse final**: el resultado se carga en SQL Server, en un modelo estrella (`fact_emergencias`, `fact_egresos`, `dim_cie10`, `dim_clima`, `dim_fecha`).
5. **Power BI** se conecta directamente al Data Warehouse para el dashboard.

El workflow de KNIME está organizado en 6 metanodos secuenciales: `01_Ingesta_Datos` → `02_datos_antes` (reporte de calidad) → `03_Limpieza_Transformacion` → `04_Carga_BD` → `05_integración` → `06_Datawarehouse_load`.

## Cómo ejecutarlo

**KNIME**
- Abrir `ProyectoFinal.knwf` en KNIME Analytics Platform.
- Ejecutar los metanodos en orden (01 a 06). Requiere tener disponibles MySQL, MongoDB, Redis y SQL Server (local o remoto, ajustando las conexiones).

**Python**
```bash
pip install pandas numpy matplotlib seaborn scikit-learn sqlalchemy pyodbc redis
jupyter notebook eda_salud_ecuador.ipynb
```
Al ejecutar solicita la contraseña de SQL Server mediante `getpass` (no está hardcodeada en el código). Si no se tiene acceso a la base, se puede establecer `USE_DB = False` para cargar desde CSV.

**Power BI**
- Abrir el archivo `.pbix`, actualizar el origen de datos apuntando al SQL Server correspondiente y refrescar.

## Resultados principales

- Solo el 79.9% de las emergencias logra emparejarse con el catálogo CIE-10; el resto corresponde a códigos propios de la adaptación ecuatoriana no presentes en el estándar internacional. Se documenta como hallazgo de calidad de datos, no como error.
- Esmeraldas, Tungurahua y Cotopaxi concentran la mayor cantidad de atenciones.
- Enero de 2024 presenta un pico atípico, cercano al doble de cualquier otro mes del periodo, pendiente de contrastar contra un posible artefacto de carga de datos.
- El diagnóstico más frecuente es "Amigdalitis aguda, no especificada", lo que sugiere que buena parte de la demanda de emergencias podría resolverse en atención primaria.
- El modelo de Isolation Forest detectó anomalías día-provincia en la serie de atenciones, cruzadas posteriormente contra las variables climáticas.

## Problemas encontrados durante el desarrollo

- El catálogo CIE-10 presentaba inconsistencias jerárquicas entre capítulos; se reconstruyó la categoría a partir del propio código en lugar de depender del campo original.
- Los nombres de provincia llegaban sin tildes desde MongoDB y con errores de tipeo puntuales; se normalizaron antes de cruzar con el clima.
- Las columnas de causa de egreso venían con código y descripción concatenados en un solo campo de texto; se separaron mediante funciones de string.
- KNIME no cuenta con un nodo nativo para Redis; esa integración se resolvió con un nodo Python embebido usando la librería `redis-py`.

## Informe completo

El detalle técnico completo (metodología, calidad de datos, resultados, limitaciones) está en `docs/Informe_IEEE.docx`.

Repositorio: https://github.com/DrosherJC/Proyecto-An-lisis-de-Datos
