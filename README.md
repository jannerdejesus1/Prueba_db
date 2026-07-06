# RiwiSupply S.A.S. - Relational Database Project

## Project Description
RiwiSupply S.A.S. distributes industrial supplies nationwide. Its data (suppliers,
products, warehouses, purchases, inventory movements) used to live in one unstandardized
Excel file, causing duplicate suppliers, inconsistent names, and unreliable reporting.
This project normalizes that data to 3NF and implements it as a relational database.

## Technologies Used
- PostgreSQL 16
- SQL (DDL / DML / DQL, views, PL/pgSQL function)
- Python (openpyxl) for data cleaning and CSV generation
- Graphviz for the Entity-Relationship Diagram

## Normalization Process
**Problems found in the raw file:** duplicate suppliers/products with inconsistent
spelling (e.g. `Aceros del Norte S.A.S` vs `ACEROS NORTE`), inconsistent categories
and city names, repeated fields on every row, and mixed granularity (purchase data
and inventory-movement data combined in a single record).

- **1NF:** every column reduced to one atomic value per cell, correct data types.
- **2NF:** city, category, supplier, warehouse and product data extracted into their
  own tables, removing partial dependencies and duplication.
- **3NF:** transitive dependencies removed (e.g. `city_name` referenced via `city_id`
  instead of living inside supplier/warehouse). The "purchase" fact was separated from
  the "inventory movement" fact to resolve the mixed-granularity problem.

**Result:** 7 tables in 3NF — `riwi_city`, `riwi_category`, `riwi_supplier`,
`riwi_warehouse`, `riwi_product`, `riwi_purchase`, `riwi_inventory_movement`.

## Database Structure

| Table | Purpose |
|---|---|
| `riwi_city` | Catalog of unique cities |
| `riwi_category` | Catalog of unique product categories |
| `riwi_supplier` | One row per real supplier, linked to its city |
| `riwi_warehouse` | One row per real warehouse, linked to its city |
| `riwi_product` | One row per real product, linked to its category |
| `riwi_purchase` | Purchases made to a supplier (PO, quantity, unit price) |
| `riwi_inventory_movement` | IN/OUT stock movements per warehouse/product, optionally linked to the purchase that originated it |

All tables/columns are named in English, prefixed with `riwi_`. PKs, FKs, `NOT NULL`
and `UNIQUE` constraints are defined in `ddl/02_create_tables.sql`.

## Entity Relationship Model
See `mer/riwisupply_erd.pdf` (or `.png`).
`riwi_city`→`riwi_supplier`/`riwi_warehouse` (1:N) · `riwi_category`→`riwi_product` (1:N)
· `riwi_supplier`/`riwi_product`→`riwi_purchase` (1:N) · `riwi_product`/`riwi_warehouse`
→`riwi_inventory_movement` (1:N) · `riwi_purchase`→`riwi_inventory_movement` (1:0..N,
only IN movements reference a purchase).

## How to Run

**1. Create the database:**
```bash
psql -U postgres -f ddl/01_create_database.sql
psql -U postgres -d bd_Janner_delahoz_garabato -f ddl/02_create_tables.sql
```

**2. Load the data** — CSV files are in `data/`. Either:
- **Client-side** (`psql`, connected to the database): run the `\copy` commands for
  each table, e.g. `\copy riwi_city (city_id, city_name) FROM 'data/riwi_city.csv' DELIMITER ',' CSV HEADER;`
  (repeat for `riwi_category`, `riwi_supplier`, `riwi_warehouse`, `riwi_product`,
  `riwi_purchase`, then `riwi_inventory_movement` with `NULL AS ''`).
- **Server-side:** run `data/03_load_data.sql`, adjusting the file paths to the CSV
  location on the server.

After loading, sync the sequences (included at the end of `03_load_data.sql`), since
IDs were inserted explicitly from the CSVs.

**3. Run the DML scripts** (`dml/`): `04_insert.sql`, `05_update.sql`, `06_delete.sql`.

**4. Run the queries** (`queries/`): `query1` to `query6`, one per business need
(stock per product, movement detail, totals per supplier, movements per warehouse,
top purchased product, inventory value per warehouse).

**5. Extra points** (`extra/`): `07_views.sql` (2 views) and `08_stored_procedure.sql`
(`riwi_get_supplier(p_supplier_id)` — returns one supplier, or all if `NULL`).

## Execution Evidence
See `evidence/` for captured output of every script run against a live PostgreSQL 16
instance.

## Developer Information
- **Full name:** Janner de la Hoz
- **Clan:** Garabato
- **GitHub repository:** https://github.com/jannerdejesus1/Prueba_db.git
