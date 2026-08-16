# GIS Data Processing

Jupyter notebooks for moving large volumes of parcel and boundary data between
desktop GIS exports and a SQL Server spatial database.

These are working notebooks from a production data migration: reading multi-hundred-
thousand-row parcel datasets exported from ArcGIS Pro, reshaping them to match a
destination schema, and loading them in batches without exhausting memory or timing
out the connection.

## Notebooks

| Notebook | Purpose |
|----------|---------|
| [`Parcels_import.ipynb`](Parcels_import.ipynb) | Full parcel import pipeline — clean, reshape, load rows `0:25000` |
| [`import_9.ipynb`](import_9.ipynb) | The same pipeline applied to rows `200000:225000` |
| [`import_code.ipynb`](import_code.ipynb) | Imports West Virginia tax district boundaries |
| [`LA_missing_sections.ipynb`](LA_missing_sections.ipynb) | Queries PLSS sections for two Louisiana parishes to find coverage gaps |

`Parcels_import.ipynb` and `import_9.ipynb` are deliberately near-identical. The source
dataset was split into chunks and each chunk was run and verified independently, so each
notebook records the slice it handled.

## The import pipeline

Both parcel notebooks follow the same sequence:

1. **Read** an ArcGIS Pro chunk export (file geodatabase layer) plus its attribute CSV
2. **Drop empty geometries** — invalid features are removed before anything else touches them
3. **Join** the attribute table onto the geometry on `SourceObjectID`
4. **Reshape** — drop unused columns, rename to the destination schema, fix column order
5. **Reproject** to `EPSG:3857`
6. **Build a GeoJSON blob** per feature, stored alongside the geometry
7. **Clean nulls** — `NaN` values become empty strings so SQL Server accepts them
8. **Load in batches** of 1,000 rows with a progress bar

Step 8 is the reason for the batching: a row-by-row insert over ~900,000 parcels is
impractically slow, and a single bulk insert exceeds the statement limit. Chunking at
1,000 rows was the working compromise.

There is a size guard before loading — features whose GeoJSON exceeds 200,000 characters
are pulled out separately, because very large multipolygons overflow the destination
column.

## Requirements

- Python 3.8+ with Jupyter
- `pandas`, `geopandas`, `shapely`
- `pyodbc` and the SQL Server ODBC driver, or `pymssql` for `LA_missing_sections.ipynb`
- `sqlalchemy`, `geoalchemy2`, `tqdm`

```bash
pip install pandas geopandas shapely pyodbc pymssql sqlalchemy geoalchemy2 tqdm jupyter
```

## Configuration

Credentials are read from environment variables — nothing is hardcoded in the notebooks.

```bash
cp .env.example .env
# edit .env with your real connection details
```

| Variable | Meaning |
|----------|---------|
| `DB_SERVER` | SQL Server hostname |
| `DB_NAME` | Database name |
| `DB_USER` | Login username |
| `DB_PASSWORD` | Login password |

Load them into your shell before starting Jupyter, or use `python-dotenv` in the notebook.

## Data

Input data is not included — the source datasets are large and client-owned. `data/` and
common GIS file extensions are git-ignored so exports don't get committed by accident.

To run these against your own data you will need an ArcGIS Pro chunk export, its matching
attribute CSV, and a destination table whose schema matches the column order set in the
reshape step.
