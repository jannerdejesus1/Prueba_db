# RiwiSupply S.A.S. - Relational Database Project

## Project Description
RiwiSupply S.A.S. is a company that distributes industrial supplies nationwide.
Its operational data (suppliers, products, warehouses, purchases and inventory
movements) used to live in a single, unstandardized Excel file, causing
duplicate suppliers, inconsistent product names, repeated cities/categories,
and unreliable reporting. This project analyzes that raw data, normalizes it
up to the Third Normal Form (3NF), and implements a relational database that
supports consistent storage, referential integrity, and operational reporting.

## Technologies Used
- PostgreSQL 16
- SQL (DDL / DML / DQL, views, PL/pgSQL function)
- Python (openpyxl) for data cleaning and CSV generation
- Graphviz for the Entity-Relationship Diagram

## Database Engine Used
PostgreSQL 16.

## Normalization Process

### 1. Initial structure
The original `Dataset_RiwiSupply` Excel file contained a single flat table
(`raw_data`) with one row per inventory movement and the following columns:
`MovementDate, SupplierName, SupplierCity, Warehouse, WarehouseCity,
ProductName, Category, Quantity, UnitPrice, MovementType, PurchaseOrder`.

### 2. Problems identified
- **Duplicate suppliers with different spellings**: e.g. `Aceros del Norte
  S.A.S`, `Aceros del Norte`, `ACEROS NORTE` all refer to the same supplier.
- **Duplicate products with inconsistent names**: e.g. `Disco de Corte 4.5`
  vs `Disco Corte 4.5`; `Guante Nitrilo` vs `Guantes de Nitrilo`; `Electrodo
  E6013` vs `Soldadura E6013`.
- **Inconsistent categories**: `Herramienta` vs `Herramientas`, `Consumible`
  vs `Consumibles`, `EPP` vs `Elementos Protección`.
- **City names written in different formats**: `Cartagena` / `Ctg`,
  `Barranquilla` / `Barranquila` / `B/quilla`, `Santa Marta` / `Sta Marta`.
- **Repeated/redundant fields**: city, category, supplier and warehouse
  information was repeated on every single row instead of being stored once.
- **Mixed granularity**: a single flat row combined supplier data, purchase
  order data, and warehouse movement data, mixing two different business
  facts (a purchase vs. a stock movement) in one record. Some purchase order
  numbers were even reused on unrelated OUT movements, evidencing that this
  data was not meant to be used as a single normalized entity.

### 3. Transformations applied

**1FN (First Normal Form):** every column was reduced to an atomic value
(no combined fields), and the same row/column structure was standardized so
each cell holds a single value of a single type (e.g., dates as DATE,
quantities as INTEGER, prices as NUMERIC).

**2FN (Second Normal Form):** repeating groups and partial dependencies were
removed. City, category, supplier, warehouse and product information were
extracted into their own tables because they do not depend on the full key
of a movement (product/warehouse/date), only on their own entity key. This
eliminated the supplier/category/city duplication found in the raw file.

**3FN (Third Normal Form):** transitive dependencies were removed. For
example, `city_name` no longer lives inside `riwi_supplier` or
`riwi_warehouse` directly — it is referenced through `city_id`, since city
data does not depend on the supplier/warehouse key but on the city itself.
The same applies to `category_name` inside `riwi_product`.

Additionally, the "purchase" fact (a supplier selling a product to
RiwiSupply) was separated from the "inventory movement" fact (stock entering
or leaving a warehouse). This resolves the mixed-granularity problem in the
original file and allows OUT movements (which are not purchases) to exist
without an artificial/reused purchase order number.

### 4. Final normalized result
Seven tables in 3NF: `riwi_city`, `riwi_category`, `riwi_supplier`,
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

All tables and columns are named in English and prefixed with `riwi_`, as
required. Primary keys, foreign keys, `NOT NULL` and `UNIQUE` constraints are
defined in `ddl/02_create_tables.sql`.

## Entity Relationship Model
See `mer/riwisupply_erd.pdf` (and `mer/riwisupply_erd.png`) for the diagram.

Relationships:
- `riwi_city` (1) → (N) `riwi_supplier`
- `riwi_city` (1) → (N) `riwi_warehouse`
- `riwi_category` (1) → (N) `riwi_product`
- `riwi_supplier` (1) → (N) `riwi_purchase`
- `riwi_product` (1) → (N) `riwi_purchase`
- `riwi_product` (1) → (N) `riwi_inventory_movement`
- `riwi_warehouse` (1) → (N) `riwi_inventory_movement`
- `riwi_purchase` (1) → (0..N) `riwi_inventory_movement` (a movement is
  linked to a purchase only when it represents stock coming in from that
  purchase; OUT movements have no purchase reference)

## Instructions to Create the Database
1. Have PostgreSQL 16 (or compatible) installed and running.
2. Run the scripts in order:
   ```bash
   psql -U postgres -f ddl/01_create_database.sql
   psql -U postgres -d bd_Janner_delahoz_garabato -f ddl/02_create_tables.sql
   ```

## Instructions to Load Data
Cleaned, normalized CSV files are provided in `data/`. Two options:

**Option A — psql client-side `\copy` (no server file-system access needed):**
```bash
psql -U postgres -d bd_Janner_delahoz_garabato
\copy riwi_city (city_id, city_name) FROM 'data/riwi_city.csv' DELIMITER ',' CSV HEADER;
\copy riwi_category (category_id, category_name) FROM 'data/riwi_category.csv' DELIMITER ',' CSV HEADER;
\copy riwi_supplier (supplier_id, supplier_name, city_id) FROM 'data/riwi_supplier.csv' DELIMITER ',' CSV HEADER;
\copy riwi_warehouse (warehouse_id, warehouse_name, city_id) FROM 'data/riwi_warehouse.csv' DELIMITER ',' CSV HEADER;
\copy riwi_product (product_id, product_name, category_id) FROM 'data/riwi_product.csv' DELIMITER ',' CSV HEADER;
\copy riwi_purchase (purchase_id, purchase_order_number, supplier_id, product_id, purchase_date, quantity, unit_price) FROM 'data/riwi_purchase.csv' DELIMITER ',' CSV HEADER;
\copy riwi_inventory_movement (movement_id, movement_date, warehouse_id, product_id, movement_type, quantity, unit_price, purchase_id) FROM 'data/riwi_inventory_movement.csv' DELIMITER ',' CSV HEADER NULL AS '';
```

**Option B — server-side `COPY`:** use `data/03_load_data.sql`, updating the
file paths to the absolute location of the CSV files on the server.

After loading, sync the auto-increment sequences (included at the end of
`data/03_load_data.sql`) since IDs were inserted explicitly from the CSVs.

This strategy was chosen (over a raw SQL INSERT script) because it keeps the
cleaning logic in Python (reproducible, auditable) and the loading step in
SQL (fast, native to the database engine), while proving that the 3NF
structure can store the exact same business data without redundancy or
inconsistency.

## Data Manipulation Scripts (`dml/`)
- `04_insert.sql` — registers a new supplier ("Ferreteria del Sinu S.A.S.")
  and a product associated with it through a new purchase order.
- `05_update.sql` — updates the trade name and city of an existing supplier.
- `06_delete.sql` — deletes a product with no inventory movements, and shows
  (commented out) that deleting a product still in use is rejected by the
  foreign key constraints.

## SQL Queries (`queries/`)

| File | Business need |
|---|---|
| `query1_stock_per_product.sql` | Available stock per product (IN − OUT), for inventory planning. |
| `query2_movements_detail.sql` | Full detail of every movement with its product and warehouse, for logistics supervision. |
| `query3_total_by_supplier.sql` | Total units and amount purchased per supplier, for purchasing management. |
| `query4_movements_per_warehouse.sql` | Count of movements per warehouse, to identify the busiest warehouses. |
| `query5_top_purchased_product.sql` | The product with the highest purchased volume, to identify top rotation. |
| `query6_inventory_value_per_warehouse.sql` | Current economic value of the inventory held in each warehouse. |

> Note: because the sample dataset only covers a few months of movements and
> does not include an opening inventory balance, some computed stock/value
> figures are negative in this demo run. In a production load, an opening
> balance per product/warehouse would be loaded first.

## Extra Points (`extra/`)
- `07_views.sql` — `riwi_vw_stock_per_product` (operative) and
  `riwi_vw_purchases_per_supplier` (analytical).
- `08_stored_procedure.sql` — `riwi_get_supplier(p_supplier_id)`: returns one
  supplier when an id is passed, or all suppliers when `NULL`/no argument is
  passed. Implemented as a PL/pgSQL table function, PostgreSQL's native
  mechanism for returning result sets from a stored routine.

## Execution Evidence
See `evidence/` for the captured output of every script run against a live
PostgreSQL 16 instance (DDL creation, data load, DML operations, the 6
queries, the 2 views and the stored function).

## Developer Information
- **Full name:** Janner de la Hoz
- **Clan:** Garabato
- **GitHub repository:** https://github.com/jannerdejesus1/Prueba_db.git
