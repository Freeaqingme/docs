---
title: CREATE TABLE
summary: The CREATE TABLE statement creates a new table in a database.
toc: false
---

<script>
$(document).ready(function(){

  var $filter_button = $('.filter-button');

    $filter_button.on('click', function(){
      var scope = $(this).data('scope'),
      $current_tab = $('.filter-button.current'), $current_content = $('.filter-content.current');

      //remove current class from tab and content
      $current_tab.removeClass('current');
      $current_content.removeClass('current');

      //add current class to clicked button and corresponding content block
      $('.filter-button[data-scope="'+scope+'"').addClass('current');
      $('.filter-content[data-scope="'+scope+'"').addClass('current');
    });
});
</script>

<style>
.filters .scope-button {
  width: 12%;
}
</style>

The `CREATE TABLE` [statement](sql-statements.html) creates a new table in a database.

By default, tables are created in the default replication zone but can be placed into a specific replication zone. See [Create a Replication Zone for a Table](configure-replication-zones.html#create-a-replication-zone-for-a-table) for more information.

<div id="toc"></div>

## Required Privileges

The user must have the `CREATE` [privilege](privileges.html) on the parent database. 

## Synopsis

<div id="step-three-filters" class="filters clearfix">
  <button class="filter-button scope-button current" data-scope="basic">Basic</button>
  <button class="filter-button scope-button" data-scope="expanded">Expanded</button>
</div><p></p>

<div class="filter-content current" markdown="1" data-scope="basic">
{% include sql/diagrams/create_table.html %}
</div>

<div class="filter-content" markdown="1" data-scope="expanded">

{% include sql/diagrams/create_table.html %}

**column_def ::=**
{% include sql/diagrams/column_def.html %}

**col_qual_list ::=**
{% include sql/diagrams/col_qual_list.html %}

**index_def ::=**
{% include sql/diagrams/index_def.html %}

**family_def ::=**
{% include sql/diagrams/family_def.html %}

**table_constraint ::=**
{% include sql/diagrams/table_constraint.html %}
</div>

## Parameters

Parameter | Description
----------|------------
`IF NOT EXISTS` | Create a new table only if a table of the same name does not already exist in the database; if one does exist, do not return an error.<br><br>Note that `IF NOT EXISTS` checks the table name only; it does not check if an existing table has the same columns, indexes, constraints, etc., of the new table.
`any_name` | The name of the table to create, which must be unique within its database and follow these [identifier rules](keywords-and-identifiers.html#identifiers). When the parent database is not set as the default, the name must be formatted as `database.name`.<br><br>The [`UPSERT`](upsert.html) and [`INSERT ON CONFLICT`](insert.html) statements use a temporary table called `excluded` to handle uniqueness conflicts during execution. It's therefore not recommended to use the name `excluded` for any of your tables.
`column_def` | A comma-separated list of column definitions. Each column requires a [name/identifier](keywords-and-identifiers.html#identifiers) and [data type](data-types.html); optionally, a [column-level constraint](constraints.html) can be specified. Column names must be unique within the table but can have the same name as indexes or constraints.<br><br>Any `PRIMARY KEY`, `UNIQUE`, and `CHECK` [constraints](constraints.html) defined at the column level are moved to the table level as part of the table's creation. Use the `SHOW CREATE TABLE` statement to view them at the table level.
`index_def` | An optional, comma-separated list of [index definitions](indexes.html). For each index, the column(s) to index must be specified; optionally, a name can be specified. Index names must be unique within the table and follow these [identifier rules](keywords-and-identifiers.html#identifiers). See the [Create a Table with Secondary Indexes](#create-a-table-with-secondary-indexes) example below.<br><br>The [`CREATE INDEX`](create-index.html) statement can be used to create an index separate from table creation.
`family_def` | An optional, comma-separated list of [column family definitions](column-families.html). Column family names must be unique within the table but can have the same name as columns, constraints, or indexes.<br><br>A column family is a group of columns that are stored as a single key-value pair in the underlying key-value store. CockroachDB automatically groups columns into families to ensure efficient storage and performance. However, there are cases when you may want to manually assign columns to families. For more details, see [Column Families](column-families.html). 
`table_constraint` | An optional, comma-separated list of [table-level constraints](constraints.html). Constraint names must be unique within the table but can have the same name as columns, column families, or indexes.

## Examples

### Create a Table (No Primary Key Defined)

In CockroachDB, every table requires a [`PRIMARY KEY`](constraints.html#primary-key). If one is not explicitly defined, a column called `rowid` of the type `INT` is added automatically as the primary key, with the `unique_row(id)` function used to ensure that new rows always default to unique `rowid` values. The primary key is automatically indexed. 

{{site.data.alerts.callout_info}}Strictly speaking, a primary key's unique index is not created; it is derived from the key(s) under which the data is stored, so it takes no additional space. However, it appears as a normal unique index when using commands like <code>SHOW INDEX</code>.{{site.data.alerts.end}}

~~~ 
CREATE TABLE logon (
    user_id INT, 
    logon_date DATE
);

SHOW COLUMNS FROM logon;
+------------+------+-------+----------------+
|   Field    | Type | Null  |    Default     |
+------------+------+-------+----------------+
| user_id    | INT  | true  | NULL           |
| logon_date | DATE | true  | NULL           |
| rowid      | INT  | false | unique_rowid() |
+------------+------+-------+----------------+
(3 rows)

SHOW INDEX FROM logon;
+-------+---------+--------+-----+--------+-----------+---------+
| Table |  Name   | Unique | Seq | Column | Direction | Storing |
+-------+---------+--------+-----+--------+-----------+---------+
| logon | primary | true   |   1 | rowid  | ASC       | false   |
+-------+---------+--------+-----+--------+-----------+---------+
(1 row)
~~~

### Create a Table (Primary Key Defined)

In this example, we create a table with three columns. One column is the [`PRIMARY KEY`](constraints.html#primary-key), another is given the [`UNIQUE`](constraints.html#unique) constraint, and the third has no constraints. The primary key and column with the `UNIQUE` constraint are automatically indexed.

By default, CockroachDB would assign the `user_id` and `logoff_date` columns to a single column family, since they're of a fixed size, and `user_email` to its own column family, since it's unbounded. We know that `user_email` will be relatively small, however, so we use the `FAMILY` keyword to group it with the other columns. As a result, each new row in the table would correspond to a single underlying key-value pair. For more deails about how columns are assigned to column families, see [Column Families](column-families.html).

~~~ 
CREATE TABLE logoff (
    user_id INT PRIMARY KEY, 
    user_email STRING UNIQUE, 
    logoff_date DATE,
    FAMILY f1 (user_id, user_email, logoff_date)
);

SHOW COLUMNS FROM logoff;
+-------------+------------+-------+---------+
|    Field    |    Type    | Null  | Default |
+-------------+------------+-------+---------+
| user_id     | INT        | false | NULL    |
| user_email  | STRING     | true  | NULL    |
| logoff_date | DATE       | true  | NULL    |
+-------------+------------+-------+---------+
(3 rows)

SHOW INDEX FROM logoff;
+--------+-----------------------+--------+-----+------------+-----------+---------+
| Table  |         Name          | Unique | Seq |   Column   | Direction | Storing |
+--------+-----------------------+--------+-----+------------+-----------+---------+
| logoff | primary               | true   |   1 | user_id    | ASC       | false   |
| logoff | logoff_user_email_key | true   |   1 | user_email | ASC       | false   |
+--------+-----------------------+--------+-----+------------+-----------+---------+
(2 rows)
~~~

### Create a Table With Secondary Indexes

In this example, we create two secondary indexes during table creation. Secondary indexes allow efficient access to data with keys other than the primary key. This example also demonstrates a number of column-level and table-level [constraints](constraints.html).

~~~ 
CREATE TABLE product_information (
    product_id           INT PRIMARY KEY NOT NULL,
    product_name         STRING(50) UNIQUE NOT NULL,
    product_description  STRING(2000),
    category_id          STRING(1) NOT NULL CHECK (category_id IN ('A','B','C')),
    weight_class         INT,
    warranty_period      INT CONSTRAINT valid_warranty CHECK (warranty_period BETWEEN 0 AND 24),
    supplier_id          INT,
    product_status       STRING(20),
    list_price           DECIMAL(8,2),
    min_price            DECIMAL(8,2),
    catalog_url          STRING(50) UNIQUE,
    date_added           DATE DEFAULT CURRENT_DATE(),
    CONSTRAINT price_check CHECK (list_price >= min_price),
    INDEX date_added_idx (date_added),
    INDEX supp_id_prod_status_idx (supplier_id, product_status)
);

SHOW INDEX FROM product_information;
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+
|        Table        |                 Name                 | Unique | Seq |     Column     | Direction | Storing |
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+
| product_information | primary                              | true   |   1 | product_id     | ASC       | false   |
| product_information | product_information_product_name_key | true   |   1 | product_name   | ASC       | false   |
| product_information | product_information_catalog_url_key  | true   |   1 | catalog_url    | ASC       | false   |
| product_information | date_added_idx                       | false  |   1 | date_added     | ASC       | false   |
| product_information | supp_id_prod_status_idx              | false  |   1 | supplier_id    | ASC       | false   |
| product_information | supp_id_prod_status_idx              | false  |   2 | product_status | ASC       | false   |
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+
(6 rows)
~~~

We also have other resources on indexes:

- Create indexes for existing tables using [`CREATE INDEX`](create-index.html).
- [Learn more about indexes](indexes.html).

### Create a Table with Foreign Keys

[Foreign keys](constraints.html#foreign-keys) guarantee a column uses only values that already exist in the column it references, which must be from another table. This constraint enforces referential integrity between the two tables.

There are a [number of rules](constraints.html#rules-for-creating-foreign-keys) that govern foreign keys, but the two most important are:

- Foreign key columns must be [indexed](indexes.html) when creating the table using `INDEX`, `PRIMARY KEY`, or `UNIQUE`.
- Referenced columns must contain only unique values. This means the `REFERENCES` clause must use exactly the same columns as a [`PRIMARY KEY`](constraints.html#primary-key) or [`UNIQUE`](constraints.html#unique) constraint.

In this example, we'll show a series of tables using different formats of foreign keys.

~~~ 
CREATE TABLE customers (id INT PRIMARY KEY, email STRING UNIQUE);

CREATE TABLE products (sku STRING PRIMARY KEY, price DECIMAL(9,2));

CREATE TABLE orders (
  id INT PRIMARY KEY,
  product STRING NOT NULL REFERENCES products,
  quantity INT,
  customer INT NOT NULL CONSTRAINT valid_customer REFERENCES customers (id),
  CONSTRAINT id_customer_unique UNIQUE (id, customer),
  INDEX (product),
  INDEX (customer)
);

CREATE TABLE reviews (
  id INT PRIMARY KEY,
  product STRING NOT NULL REFERENCES products,
  customer INT NOT NULL,
  "order" INT NOT NULL,
  body STRING,
  CONSTRAINT order_customer_fk FOREIGN KEY ("order", customer) REFERENCES orders (id, customer),
  INDEX (product),
  INDEX (customer),
  INDEX ("order", customer)
);
~~~

### Show the Definition of a Table

To show the definition of a table, use the `SHOW CREATE TABLE` statement. The contents of the `CreateTable` column in the response is a string with embedded line breaks that, when echoed, produces formatted output.

~~~ 
SHOW CREATE TABLE logoff;
+--------+----------------------------------------------------------+
| Table  |                       CreateTable                        |
+--------+----------------------------------------------------------+
| logoff | CREATE TABLE logoff (␤                                   |
|        |     user_id INT NOT NULL,␤                               |
|        |     user_email STRING(50) NULL,␤                         |
|        |     logoff_date DATE NULL,␤                              |
|        |     CONSTRAINT "primary" PRIMARY KEY (user_id),␤         |
|        |     UNIQUE INDEX logoff_user_email_key (user_email),␤    |
|        |     FAMILY "primary" (user_id, user_email, logoff_date)␤ |
|        | )                                                        |
+--------+----------------------------------------------------------+
(1 row)
~~~

## See Also

- [`INSERT`](insert.html)
- [`ALTER TABLE`](alter-table.html)
- [`DELETE`](delete.html)
- [`DROP TABLE`](drop-table.html)
- [`RENAME TABLE`](rename-table.html)
- [`SHOW TABLES`](show-tables.html)
- [`SHOW COLUMNS`](show-columns.html)
- [Column Families](column-families.html)
- [Table-Level Replication Zones](configure-replication-zones.html#create-a-replication-zone-for-a-table)
