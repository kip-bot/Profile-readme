# Postgres GIN Index in Rails

In a Rails application, adding an index to any database column is just a one-liner migration.

```rb
add_index :table_name, :column_name
```
While learning more about Postgres database indexes, you would have come across the fact that Postgres provides several index types, from which `B-Tree` is the most famous one. 

But do you know what type of index is used in the migration above? Postgres by default applies the `B-Tree` index, unless specified explicitly. 

Other than the `B-Tree` index, there is one more most useful index type `GIN` (Generalized Inverted Index). Here in this article, we are going to look at the `GIN` index and how it's used in Postgres databases. A question may come to your mind, what is special about the `GIN` index?

> GIN indexes are the perfect choice for “composite values” where you perform a query that looks for an element within such “composite” columns like jsonb, array or hstore data structures. 

### GIN index for text columns:

These days most applications have a search feature. Searching with small string columns which store single values like name, city, etc works fine with the B-Tree index. But when it comes to full-text searching with text type columns, is the B-Tree index that helpful? Let's try to find out.

Consider the following example, here we are creating `messages` table, having `recipient_id` and `message_body` columns and adding an index on the `message_body` column. 

```sql
CREATE TABLE messages (recipient_id INTEGER, message_body TEXT);
CREATE INDEX idx_message_body ON messages(message_body);
```

Since we have not provided any index type, it will create a B-Tree index. Let's verify that here:

```sql
SELECT
    indexname,
    indexdef
FROM
    pg_indexes
WHERE
    tablename = 'messages';
```

![Screenshot 2021-11-26 at 12 01 37 PM](https://user-images.githubusercontent.com/1950768/143555889-fb81d527-7d0b-4830-86dc-77bc26093b53.png)

Now let's add a row with a lengthy message body here.

```sql
INSERT INTO messages(recipient_id, message_body)
VALUES (11, repeat('Hello, I am Swati. ', 90000) )
RETURNING *;  
```

This will give an output as:

```sql
ERROR: index row requires 19616 bytes, maximum size is 8191
```

And if you are trying the same in Rails application, it will throw an error like:

```md
PG::ProgramLimitExceeded: ERROR:  index row requires 19616 bytes, maximum size is 8191
01 : CREATE INDEX "idx_message_body" ON "messages" ("message_body")
```

Are you wondering why this error was thrown? Let's first check the size of our index.

```sql
select * from pg_relation_size('idx_message_body');
```
![pg-relation-size](https://user-images.githubusercontent.com/1950768/143556204-a28433f4-2031-4f10-9082-54d98560dc2c.png)

Here the B-Tree index is not sufficient for our requirement. And hence the GIN index comes to rescue :) Let's try adding an index of type GIN.

```sql
CREATE INDEX idx_message_body ON messages USING gin (message_body gin_trgm_ops);
```

And to achieve this in Rails, you need to add migration as follows:

```rb
class AddBodyIndexOnMessages < ActiveRecord::Migration
  def up
    execute "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
    execute "CREATE INDEX idx_message_body ON messages USING gin (message_body gin_trgm_ops);"
  end

  def down
    execute "DROP INDEX idx_message_body;"
  end
end
```
![Screenshot 2021-11-26 at 11 59 03 AM](https://user-images.githubusercontent.com/1950768/143556371-9b1d9054-b473-43da-b523-69ca4d31fab5.png)

> The `pg_trgm` module provides functions and operators for determining the similarity of ASCII alphanumeric text based on trigram matching, as well as index operator classes that support fast searching for similar strings.

Let's run the query again:

```sql
EXPLAIN INSERT INTO messages(recipient_id, message_body)
VALUES (11, repeat('Hello, I am Swati. ', 90000) )
RETURNING *;  
```

![Screenshot 2021-11-26 at 11 51 39 AM](https://user-images.githubusercontent.com/1950768/143556511-69de435a-e6dc-419b-88f7-3da59d9b19a1.png)

Let's try searching the message body with the following query. Now, if you execute the query, you will find that the database engine uses the index for lookup:

```sql
EXPLAIN SELECT *
FROM messages
WHERE message_body ILIKE '%Swa%';  
```
![Screenshot 2021-11-26 at 12 04 07 PM](https://user-images.githubusercontent.com/1950768/143556569-a5082740-af42-4a84-9f37-167f7ed3ecfa.png)

This is how we can use GIN index in our application to search values in text columns.

### GIN index for array columns:

Let's look into how we can use GIN index for the array data structure. Take an example of companies database storing multiple contact numbers.

```sql
CREATE TABLE companies (company_code INTEGER, contact_numbers TEXT []);
CREATE INDEX idx_contact_numbers ON companies USING gin (contact_numbers);
```

To search company records by matching contact number: 

```sql
EXPLAIN SELECT *
FROM companies
WHERE contact_numbers @> ARRAY['(408)-743-9045'];
``` 

![Screenshot 2021-11-26 at 1 38 22 PM](https://user-images.githubusercontent.com/1950768/143556751-da3b258c-5279-44ea-82eb-af130ae13ed0.png)


This is how the GIN index helps us to apply search on the composite columns. 

Hope this article helps you to get introduced to GIN index of Postgres. 


### References:

- [Indexes on Rails application](https://karolgalanciak.com/blog/2018/08/19/indexes-on-rails-how-to-make-the-most-of-your-postgres-database/)
- [PostgreSQL Doc: GiST and GIN Index Types](https://www.postgresql.org/docs/9.1/textsearch-indexes.html)
