### Q1. How do you find the second-highest salary from an employee table?

```sql
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);
```
### Q2. How do you add a new column to a database table containing millions of records while the application is running 24/7 without downtime?

**Answer:**

1. Add a new nullable column using `ALTER TABLE` to minimise the initial schema-change impact.
2. Update the application code to populate the new column for all newly inserted records.
3. Deploy the application code.
4. Backfill existing records in small batches, for example, 500 rows at a time.
5. Monitor database load, performance and application health between batches.
6. Continue the batch updates until all existing records are populated.

```sql
ALTER TABLE employees ADD COLUMN new_column VARCHAR(100) NULL;
```
