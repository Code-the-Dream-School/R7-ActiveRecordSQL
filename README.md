# SQL in Active Record  (Enrichment Topic)

Active Record provides many methods to create/read/update/delete entries in the database.  However, some of those methods may not do exactly what you want.  Fortunately, you can use all the power of SQL.  Some of the ways to do this are described here.  In the examples below, we assume that you have two models, Customer and Order, and that there is a one-to-many association between them, where a customer has_many :orders and an order belongs_to a :customer.

## What SQL does Active Record Choose?

You can find this out as follows.  Suppose, in the case we have described, we have a customer with id 5, and we want to find all the orders for that customer.  You can do this with
```
customer = Customer.find(5)
customer.orders
```
And you can see the SQL performed as follows:
```
Customer.find(5).to_sql
customer = Customer.find(5)
customer.orders.to_sql
```

## Using Your Own SQL

One way is to use find_by_sql:
```
orders = Order.find_by_sql("SELECT orders.* FROM orders WHERE order.customer_id = 5;")
```
And you can also do this with a bind parameter, in which case you pass an array:
```
orders = Order.find_by_sql("SELECT orders.* FROM orders WHERE order.customer_id = ?;", [5])
```
Here the optional second argument is the the array of bind parameters.

You can make the SQL as complicated as you need, with JOIN, ORDER BY, LIMIT, HAVING, etc.  However, find_by_sql is a method, in this case, of the Order model, so what you retrieve is an Active Record relation of Orders.  You have to be careful to select the attributes corresponding to the Orders class.

Be very careful in passing user input to an SQL query.  This is how SQL injection attacks occur.  If you pass user input as bind parameters to an SQL query, the parameters are sanitized by Active Record, preventing the attack.

## Using Your Own SQL To Return an Array of Hashes (Execute)

The following query:
```
records = ActiveRecord::Base.connection.exec_query("SELECT orders.*, customer_name FROM orders JOIN customers ON orders.customer_id = customers.id;")
```
will return an array of hashes.  Each row is returned as a hash, and in the hashes the keys are the column names from the tables.  If you select the right collection of attributes, you could pass these to a new or create operation.  You can also pass one or more bind parameters:
```
records = ActiveRecord::Base.connection.exec_query("SELECT orders.* FROM ORDERS WHERE id = ?;", "SQL", [3])
```

You can do write operations using connection.execute (INSERT INTO, UPDATE, DELETE), but this can be troublesome.  Active Record entries may have attributes like created_at that are clumsy to add.  Also, such an approach bypasses the validations in your model.  You would do write operations this way for tables that are only to be accessed using SQL, and that are never accessed using Active Directory models.

## Using Your Own SQL To Return a Result with Arrays of Values (Select All)

The following query:
```
result = ActiveRecord::Base.connection.select_all("SELECT orders.*, customer_name FROM orders JOIN customers ON orders.customer_id = customers.id;")
```
returns a Result object.  This object has two useful methods.  The result.columns method returns an array of column headers from the result of the query.  The result.rows method returns an array of arrays, where each of the inner arrays is a row of the returned values.  You could use this in combination with the Ruby CSV class to write CSV files.  Of course, you can also do this using methods from the Order model:
```
csv_string = CSV.generate do |csv|
  csv << Customer.attribute_names
  Customer.all do |customer|
    csv << customer.attributes.values
  end
end
```
The advantage of select_all is that you can easily combine information from several tables, change column names, etc.  You can also pass bind parameters using the same syntax as for exec_query.