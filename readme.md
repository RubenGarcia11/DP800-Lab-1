# Práctica 1: Diseño e implementación de objetos de base de datos con SQL





## 1. Configuración del entorno
Cloné el repositorio del laboratorio desde GitHub en Visual Studio Code (requiere SQL Server 2025+).

Usé la opción `Git: Clone` con la URL:

```
https://github.com/MicrosoftLearning/mslearn-sql-developer.git
```

![Git Clone](images/01-GIT-CLONE.png)

## 2. Creación de la base de datos
En SQL Server Management Studio (SSMS), ejecuté el siguiente script para crear la base de datos `EcommerceDB`:

```sql
 CREATE DATABASE EcommerceDB;
 GO

 USE EcommerceDB;
 GO
```

![Create Database](images/02-CREATE-DATABASE.png)

## 3. Tablas principales y restricciones
Creé las tablas de proveedores (`Supplier`), categorías (`Category`) y productos (`Product`), configurando claves y restricciones `CHECK` para garantizar la integridad de los datos.

```sql
 USE EcommerceDB;
 GO

 -- Create Supplier table
 CREATE TABLE Supplier (
     SupplierID INT PRIMARY KEY IDENTITY(1,1),
     SupplierName NVARCHAR(100) NOT NULL UNIQUE,
     Country NVARCHAR(50) NOT NULL,
     Email NVARCHAR(100),
     Phone NVARCHAR(20),
     CreatedDate DATETIME2 DEFAULT GETUTCDATE()
 );

 -- Create Category table
 CREATE TABLE Category (
     CategoryID INT PRIMARY KEY IDENTITY(1,1),
     CategoryName NVARCHAR(100) NOT NULL UNIQUE,
     Description NVARCHAR(500)
 );

 -- Create Product table with constraints
 CREATE TABLE Product (
     ProductID INT PRIMARY KEY IDENTITY(1,1),
     ProductName NVARCHAR(100) NOT NULL,
     CategoryID INT NOT NULL,
     SupplierID INT NOT NULL,
     BasePrice DECIMAL(10,2) NOT NULL,
     StockQuantity INT NOT NULL DEFAULT 0,
     CreatedDate DATETIME2 DEFAULT GETUTCDATE(),
     CHECK (BasePrice > 0),
     CHECK (StockQuantity >= 0),
     FOREIGN KEY (CategoryID) REFERENCES Category(CategoryID),
     FOREIGN KEY (SupplierID) REFERENCES Supplier(SupplierID)
 );

 -- Create indexes
 CREATE INDEX IX_Category ON Product(CategoryID);
 CREATE INDEX IX_Supplier ON Product(SupplierID);
 GO
```

![Core Tables](images/03-CORE-TABLES.png)

Luego inserté datos de prueba para verificar el esquema.

```sql
 USE EcommerceDB;
 GO

 -- Insert sample suppliers
 INSERT INTO Supplier (SupplierName, Country, Email, Phone)
 VALUES 
     ('Contoso Supplies', 'USA', 'contact@contoso.com', '555-0100'),
     ('Fabrikam Inc', 'Canada', 'sales@fabrikam.com', '555-0200');

 -- Insert sample categories
 INSERT INTO Category (CategoryName, Description)
 VALUES 
     ('Electronics', 'Electronic devices and accessories'),
     ('Clothing', 'Apparel and fashion items');

 -- Insert sample products
 INSERT INTO Product (ProductName, CategoryID, SupplierID, BasePrice, StockQuantity)
 VALUES 
     ('Wireless Mouse', 1, 1, 29.99, 100),
     ('Cotton T-Shirt', 2, 2, 19.99, 250);
 GO
```

![Samples](images/04-SAMPLES.png)

## 4. Tabla temporal (Historial de precios)
Para registrar el historial de precios, creé la tabla `ProductPrice` con control de versiones (`SYSTEM_VERSIONING`).

```sql
 USE EcommerceDB;
 GO

 -- Create Price History table with temporal versioning
 CREATE TABLE ProductPrice (
     PriceID INT PRIMARY KEY IDENTITY(1,1),
     ProductID INT NOT NULL,
     CurrentPrice DECIMAL(10,2) NOT NULL,
     EffectiveDate DATE,
     SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN,
     SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END HIDDEN,
     PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime),
     FOREIGN KEY (ProductID) REFERENCES Product(ProductID)
 ) WITH (SYSTEM_VERSIONING = ON);
 GO

 -- Insert initial price data
 INSERT INTO ProductPrice (ProductID, CurrentPrice, EffectiveDate)
 VALUES (1, 99.99, '2025-01-01'), (2, 149.99, '2025-01-01');

 -- Update price (creates history entry)
 UPDATE ProductPrice SET CurrentPrice = 109.99 WHERE ProductID = 1;
 GO
```

![Temporal Table](images/05-TEMP-TABLE.png)

Consultando con `FOR SYSTEM_TIME ALL`, el sistema muestra el precio actual y los anteriores con sus fechas.

```sql
 USE EcommerceDB;
 GO

 -- Query price history
 SELECT ProductID, CurrentPrice, SysStartTime, SysEndTime
 FROM ProductPrice
 FOR SYSTEM_TIME ALL
 WHERE ProductID = 1;
```

![Query](images/06-QUERY.png)

## 5. Columnas JSON para metadatos
Añadí una columna JSON a la tabla `Product` para almacenar propiedades variables, junto con una columna calculada para el "color" y un índice para agilizar búsquedas.

```sql
 USE EcommerceDB;
 GO

 -- Add metadata column to Product (JSON type requires SQL Server 2025)
 ALTER TABLE Product ADD Metadata JSON;
 GO

 -- Add computed column for indexing
 ALTER TABLE Product ADD MetadataColor AS JSON_VALUE(Metadata, '$.color');
 GO

 -- Create index on the computed column
 CREATE NONCLUSTERED INDEX IX_Product_Metadata_Color
     ON Product (MetadataColor);
 GO

 -- Update products with metadata
 UPDATE Product SET Metadata = N'{"color":"blue","size":"large","material":"cotton"}'
 WHERE ProductID = 1;

 UPDATE Product SET Metadata = N'{"color":"red","size":"small","material":"silk"}'
 WHERE ProductID = 2;
 GO
```

![JSON Column](images/07-JSON-COLUMN.png)

Luego, extraje propiedades específicas usando `JSON_VALUE`.

```sql
 USE EcommerceDB;
 GO

 -- Query JSON data
 SELECT 
     ProductID,
     ProductName,
     JSON_VALUE(Metadata, '$.color') AS Color,
     JSON_VALUE(Metadata, '$.size') AS Size,
     JSON_VALUE(Metadata, '$.material') AS Material
 FROM Product
 WHERE JSON_VALUE(Metadata, '$.color') = 'blue';
```

![Query JSONs](images/08-QUERY-JSONS.png)

## 6. Tabla de pedidos particionada
Para mejorar el rendimiento al manejar grandes volúmenes de datos, particioné la tabla `Order` por trimestres.

```sql
 USE EcommerceDB;
 GO

 -- Create partition function for order dates
 -- Use RANGE RIGHT for date columns to keep same-day values together
 CREATE PARTITION FUNCTION PF_OrderDate (DATE)
     AS RANGE RIGHT FOR VALUES 
     ('2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01');

 -- Create partition scheme (single filegroup recommended)
 CREATE PARTITION SCHEME PS_OrderDate
     AS PARTITION PF_OrderDate ALL TO ([PRIMARY]);

 -- Create partitioned Order table
 -- Include OrderDate in primary key for clustered index alignment
 CREATE TABLE [Order] (
     OrderID BIGINT IDENTITY(1,1),
     OrderDate DATE NOT NULL,
     CustomerName NVARCHAR(100) NOT NULL,
     TotalAmount DECIMAL(12,2) NOT NULL,
     OrderStatus NVARCHAR(20) DEFAULT 'Pending',
     CONSTRAINT PK_Order PRIMARY KEY (OrderID, OrderDate),
     CHECK (TotalAmount > 0),
     CHECK (OrderStatus IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled'))
 ) ON PS_OrderDate(OrderDate);

 -- Create partitioned index
 CREATE NONCLUSTERED INDEX IX_Order_Customer
     ON [Order](CustomerName)
     ON PS_OrderDate(OrderDate);
 GO

 -- Insert sample orders
 INSERT INTO [Order] (OrderDate, CustomerName, TotalAmount, OrderStatus) VALUES
     ('2025-01-15', 'John Smith', 299.97, 'Delivered'),
     ('2025-02-20', 'Jane Doe', 149.99, 'Shipped'),
     ('2025-06-10', 'Bob Johnson', 449.95, 'Processing');
 GO
```

![Partitioned Order Table](images/09-PARTITIONED-ORDER-TABLE.png)

Comprobé con `$PARTITION` que los registros se distribuyen correctamente por fecha.

```sql
 USE EcommerceDB;
 GO

 -- Query by partition
 SELECT 
     $PARTITION.PF_OrderDate(OrderDate) AS PartitionNumber,
     COUNT(*) AS OrdersInPartition,
     MIN(OrderDate) AS MinDate,
     MAX(OrderDate) AS MaxDate
 FROM [Order]
 GROUP BY $PARTITION.PF_OrderDate(OrderDate);
```

![Query Partitioned Table](images/10-QUERY-PARTITIONED-TABLE.png)

## 7. Identificadores con SEQUENCE
Usé un objeto `SEQUENCE` independiente para generar los IDs de las líneas de pedido (`OrderDetail`).

```sql
 USE EcommerceDB;
 GO

 -- Create SEQUENCE for order line items
 CREATE SEQUENCE OrderLineSequence
     START WITH 1
     INCREMENT BY 1;

 -- Create OrderDetail table
 CREATE TABLE OrderDetail (
     OrderLineID INT PRIMARY KEY,
     OrderID BIGINT NOT NULL,
     OrderDate DATE NOT NULL,
     ProductID INT NOT NULL,
     Quantity INT NOT NULL,
     UnitPrice DECIMAL(10,2) NOT NULL,
     LineTotal AS (Quantity * UnitPrice),
     CHECK (Quantity > 0),
     CHECK (UnitPrice > 0),
     FOREIGN KEY (OrderID, OrderDate) REFERENCES [Order](OrderID, OrderDate),
     FOREIGN KEY (ProductID) REFERENCES Product(ProductID)
 );
 GO

 -- Insert order details using SEQUENCE
 INSERT INTO OrderDetail (OrderLineID, OrderID, OrderDate, ProductID, Quantity, UnitPrice)
 VALUES 
     (NEXT VALUE FOR OrderLineSequence, 1, '2025-01-15', 1, 2, 99.99),
     (NEXT VALUE FOR OrderLineSequence, 1, '2025-01-15', 2, 1, 149.99),
     (NEXT VALUE FOR OrderLineSequence, 2, '2025-02-20', 1, 3, 99.99);
 GO
```

![Sequence](images/11-SEQUENCE.png)

Verifiqué que los números se generaron de forma correlativa.

```sql
 USE EcommerceDB;
 GO

 SELECT * FROM OrderDetail;
```

![Verify Sequence](images/12-VERIFY-SEQUENCE.png)

## 8. Verificación final
Intenté insertar un precio negativo para comprobar que la restricción `CHECK` bloquea el error correctamente.

```sql
 USE EcommerceDB;
 GO

 -- Verify constraints work
 -- This should fail: negative price
 INSERT INTO Product (ProductName, CategoryID, SupplierID, BasePrice, StockQuantity)
 VALUES ('Invalid', 1, 1, -50, 10);
```

![Query Fail](images/13-QUERY-FAIL.png)

Por último, realicé consultas finales de validación sobre JSON, particiones e historial temporal.

```sql
 USE EcommerceDB;
 GO

 -- Verify JSON queries work
 SELECT ProductName, JSON_VALUE(Metadata, '$.color') AS Color
 FROM Product
 WHERE Metadata IS NOT NULL;

 -- Verify partitioning
 SELECT $PARTITION.PF_OrderDate(OrderDate) AS Partition, COUNT(*) AS RecordCount
 FROM [Order]
 GROUP BY $PARTITION.PF_OrderDate(OrderDate);

 -- Verify temporal table
 SELECT ProductID, CurrentPrice, SysStartTime, SysEndTime
 FROM ProductPrice FOR SYSTEM_TIME ALL
 ORDER BY ProductID, SysStartTime;
```

![Verify Partitioning Table](images/14-VERIFY-PARTITIONING-TABLE.png)

