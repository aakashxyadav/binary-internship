**Part 1: Code Review & Debugging
**
1. Issues Identified


A. Missing Input Validation

Issue

The code directly accesses data['field'] without checking if
- request.json exists
- Required fields are present
- Data types are correct

Impact in Production

- API will crash with KeyError if a field is missing
- Invalid data (negative price, string quantity) can be stored
- Leads to unstable API behavior and poor client experience


B. SKU Uniqueness Not garanteed 

- No check to ensure sku is unique

Impact in Production

- Duplicate SKUs break inventory tracking

C. Transaction is not atomic

- Two separate commit() calls
- If inventory creation fails, product remains created

Impact in Production
- Partial data persists
- Products exist without inventory

D. Prices type not defined 

- Price is accepted without validation or type handling
- Likely stored as float instead of decimal

Impact in Production

- Financial inaccuracies over time
- Incorrect billing or reporting

E. No Error Handling 
-  No try/except

Impact in Production
- Silent failures or 500 errors
- Database left in inconsistent state
- Harder debugging and monitoring

F. Incorrect Warehouse Assumption

- The endpoint assumes a product belongs to only one warehouse
- Inventory is created immediately for one warehouse

Impact in Production
- Conflicts with the requirement that products can exist in multiple warehouses
- Makes future warehouse expansion harder
- Forces unnecessary duplication of product records


Corrected Code :

```python
from decimal import Decimal
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # 1. Basic input validation
    if not data:
        return {"error": "Invalid JSON payload"}, 400

    required_fields = ['name', 'sku', 'price', 'warehouse_id']
    for field in required_fields:
        if field not in data:
            return {"error": f"{field} is required"}, 400

    # 2. Validate price
    try:
        price = Decimal(str(data['price']))
        if price < 0:
            return {"error": "Price must be non-negative"}, 400
    except:
        return {"error": "Invalid price format"}, 400

    try:
        # 3. Create product
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price
        )

        db.session.add(product)
        db.session.flush()  # get product.id without committing

        # 4. Create inventory for warehouse
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data.get('initial_quantity', 0)
        )

        db.session.add(inventory)

        # 5. Single commit to keep transaction atomic
        db.session.commit()

        return {
            "message": "Product created successfully",
            "product_id": product.id
        }, 201

    except IntegrityError:
        db.session.rollback()
        return {"error": "SKU must be unique"}, 409

    except Exception:
        db.session.rollback()
        return {"error": "Something went wrong"}, 500


```

Explaination of fixes:

Input Validation

I added basic checks to make sure the request body exists and required fields are present. Earlier, the API assumed everything would be sent correctly, which could easily cause crashes if a field was missing.

Price Validation

I validated the price to ensure it is a valid number and not negative. Since price is used for calculations and reporting, saving invalid values could create problems later.

Atomic Transaction

The original code committed the product before creating inventory. I changed this so both product and inventory are saved together. This avoids cases where a product exists without inventory.

Using flush() Instead of Early Commit

I used flush() to get the product ID without committing it immediately. This helps keep the database consistent if something fails later.

Error Handling

I wrapped database operations in a try-except block and added rollback on failure. This ensures the database does not end up in a partial or broken state.

SKU Uniqueness Handling

I handled duplicate SKU errors explicitly. Since SKU is a key business identifier, this prevents multiple products from being created with the same SKU.

Warehouse Handling

I kept the existing warehouse logic and did not redesign it. Larger changes around warehouse relationships would need more business clarity and senior input.


**Part 2: Database Design
**

Companies

```sql
CREATE TABLE companies (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

```

Warehouses

```sql

CREATE TABLE warehouses (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_warehouse_company
        FOREIGN KEY (company_id) REFERENCES companies(id)
);
```

Products

```sql

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_product_sku UNIQUE (sku)
);

```
 Inventory 

```sql
CREATE TABLE inventory (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    CONSTRAINT fk_inventory_product
        FOREIGN KEY (product_id) REFERENCES products(id),

    CONSTRAINT fk_inventory_warehouse
        FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),

    CONSTRAINT uq_product_warehouse UNIQUE (product_id, warehouse_id)
);
```
 Inventory History (Track Quantity Changes)

```sql
CREATE TABLE inventory_history (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    old_quantity INT,
    new_quantity INT,
    reason VARCHAR(255),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_history_product
        FOREIGN KEY (product_id) REFERENCES products(id),

    CONSTRAINT fk_history_warehouse
        FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);
```

Suppliers

```sql
CREATE TABLE suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    contact_info VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

 Supplier to Product Mapping (Many-to-Many)

```sql
CREATE TABLE supplier_products (
    supplier_id INT NOT NULL,
    product_id INT NOT NULL,

    PRIMARY KEY (supplier_id, product_id),

    CONSTRAINT fk_supplier_product_supplier
        FOREIGN KEY (supplier_id) REFERENCES suppliers(id),

    CONSTRAINT fk_supplier_product_product
        FOREIGN KEY (product_id) REFERENCES products(id)
);
```

Product Bundles (Self-referencing Products)

```sql
CREATE TABLE product_bundles (
    bundle_id INT NOT NULL,
    child_product_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,

    PRIMARY KEY (bundle_id, child_product_id),

    CONSTRAINT fk_bundle_parent
        FOREIGN KEY (bundle_id) REFERENCES products(id),

    CONSTRAINT fk_bundle_child
        FOREIGN KEY (child_product_id) REFERENCES products(id)
);
```
2.  Questions for Product Team

Before finalizing this design, I would clarify the following:
1. Can a product belong to multiple companies or only one?
2. Should inventory changes be tracked automatically or only for certain actions?
3. Do suppliers provide products globally or per company?
4. Is negative inventory allowed (backorders)?
5. Should inventory history store the user/system that made the change?

3. Design Decisions Explained 

Inventory as a Separate Table
Inventory is warehouse-specific, so keeping it separate from products avoids duplication and supports multi-warehouse storage.

Inventory History Table
Instead of overwriting quantities, changes are logged. This helps with auditing, debugging stock issues, and future reporting.

Product Bundles Using a Mapping Table
Using a self-referencing table allows flexible bundles and avoids hardcoding bundle logic inside the product table.

Indexes & Constraints
Unique index on products.sku to prevent duplicates
Composite unique key on (product_id, warehouse_id) in inventory
Foreign keys to maintain referential integrity


4. Why This Design Works
Supports multiple warehouses per company
Handles products in multiple warehouses
Tracks inventory changes cleanly
Supports suppliers and bundled products
Easy to scale without major redesign

** Part 3: API Implementation 
**
1. Assumptions 
Since requirements are incomplete, I made the following reasonable assumptions:
Each product has a product_type (used to decide low-stock threshold)

Low-stock threshold is stored per product type

Recent sales activity means at least one sale in the last 30 days

Sales data exists in a sales table

A product can have multiple suppliers, but we return the primary supplier

Stockout days are estimated using average daily sales (simple approach)

2. API Implementation 

```python

@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    alerts = []

    # Get all warehouses for the company
    warehouses = Warehouse.query.filter_by(company_id=company_id).all()

    for warehouse in warehouses:
        # Get inventory records for each warehouse
        inventories = (
            db.session.query(Inventory, Product)
            .join(Product, Inventory.product_id == Product.id)
            .filter(Inventory.warehouse_id == warehouse.id)
            .all()
        )

        for inventory, product in inventories:

            # Check recent sales activity (last 30 days)
            recent_sales = (
                db.session.query(func.sum(Sale.quantity))
                .filter(
                    Sale.product_id == product.id,
                    Sale.warehouse_id == warehouse.id,
                    Sale.created_at >= datetime.utcnow() - timedelta(days=30)
                )
                .scalar()
            )

            if not recent_sales:
                continue  # Skip products with no recent sales

            # Get low stock threshold based on product type
            threshold = (
                db.session.query(ProductThreshold.threshold)
                .filter(ProductThreshold.product_type == product.product_type)
                .scalar()
            )

            if inventory.quantity >= threshold:
                continue  # Stock level is safe

            # Calculate days until stockout (simple estimation)
            avg_daily_sales = recent_sales / 30
            days_until_stockout = (
                int(inventory.quantity / avg_daily_sales)
                if avg_daily_sales > 0 else None
            )

            # Get primary supplier
            supplier = (
                db.session.query(Supplier)
                .join(SupplierProduct)
                .filter(SupplierProduct.product_id == product.id)
                .first()
            )

            alerts.append({
                "product_id": product.id,
                "product_name": product.name,
                "sku": product.sku,
                "warehouse_id": warehouse.id,
                "warehouse_name": warehouse.name,
                "current_stock": inventory.quantity,
                "threshold": threshold,
                "days_until_stockout": days_until_stockout,
                "supplier": {
                    "id": supplier.id if supplier else None,
                    "name": supplier.name if supplier else None,
                    "contact_email": supplier.contact_email if supplier else None
                }
            })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }, 200
    
```

3. Edge Cases Considered

Company has no warehouses
Warehouse has no inventory
Product has no recent sales
Product has no supplier
Average daily sales = 0 (avoid divide-by-zero)
Product type does not have a defined threshold

. Explanation of Approach 

I first fetch all warehouses for the given company since stock is warehouse-specific.

For each warehouse, I check inventory and join it with product data.

I only consider products that had sales in the last 30 days to avoid false alerts.

The low-stock threshold is picked based on product type.

If current stock is below the threshold, an alert is created.

Days until stockout is calculated using a simple average daily sales formula.

Supplier details are included to help with quick reordering.

The response groups all alerts and also returns a total count.
