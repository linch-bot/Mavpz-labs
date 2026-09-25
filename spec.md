# Data Specification (ER Model)

## Intent
The intent of this specification is to define the core data entities and their relationships for a basic online store. This model will serve as the foundation for the application's database architecture, ensuring that customer orders, products, and categories are properly tracked and normalized without using physical database schemas at this stage.

## Entities and Attributes
1. **Customer**
   - id (UUID, PK)
   - email (String)
   - full_name (String)
   - created_at (Timestamp)

2. **Product**
   - id (UUID, PK)
   - name (String)
   - price (Decimal)
   - category_id (UUID, FK)

3. **Category**
   - id (UUID, PK)
   - name (String)
   - description (String)

4. **Order**
   - id (UUID, PK)
   - customer_id (UUID, FK)
   - product_id (UUID, FK)
   - order_date (Timestamp)
   - status (String)

## Relationships
- One Customer can place many Orders.
- One Category can contain many Products.
- Each Order contains exactly one Product.

## Acceptance Criteria (for AI)
- The model must be in Mermaid erDiagram format.
- Do not use physical SQL data types; use conceptual ones.
- Keys (PK/FK) must be clearly marked.
- Strictly adhere to the relationship cardinality described above.