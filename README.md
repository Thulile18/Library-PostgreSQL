<img src="https://socialify.git.ci/Thulile18/Library-PostgreSQL/image?language=1&owner=1&name=1&stargazers=1&theme=Light" alt="Library-PostgreSQL" width="640" height="320" />

Library Management System (PostgreSQL)

A library database built in PostgreSQL to manage books, authors, and patrons — supporting adding, viewing, updating, and deleting records, plus advanced filtering queries.
Tech Stack
Database: PostgreSQL
Tools: pgAdmin 4 or `psql`
---
Actual Schema
This reflects the tables as they exist in the database (not the boolean-based version from the original brief):
```sql
CREATE TABLE author (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    nationality VARCHAR(100),
    birth_year INT,
    death_year INT
);

CREATE TABLE books (
    id INT PRIMARY KEY,
    title VARCHAR(100),
    author_id INT REFERENCES author(id),
    genres TEXT[],
    published_year INT,
    available_copies TEXT[]
);

CREATE TABLE patrons (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    book_id INT,
    borrowed_books INT[]
);
```
Key differences from the original brief:
`author` is singular (not `authors`).
Books don't have a plain `available BOOLEAN` — instead, `available_copies` is a `TEXT[]` listing the physical copies still on the shelf (e.g. `'Copy 1'`, `'Copy 2'`). A book is "available" when this array is non-empty; it's "borrowed out" when empty.
`patrons` has an extra `book_id INT` column alongside `borrowed_books INT[]`. In the current data it's `NULL` for every row — `borrowed_books` is the array actually being used to track what each patron has out. You can drop `book_id` if you don't end up using it:
```sql
  ALTER TABLE patrons DROP COLUMN book_id;
  ```
---
Sprint 1: Project Setup
```sql
CREATE DATABASE LibraryDB;
\c LibraryDB
```
Then run the `CREATE TABLE` statements above, in this order: `author` → `books` → `patrons` (order matters — `books.author_id` references `author.id`).
---
Sprint 2: Insert Data
Authors:
```sql
INSERT INTO author (id, name, nationality, birth_year, death_year) VALUES
(1, 'George Orwell', 'British', 1903, 1950),
(2, 'Harper Lee', 'American', 1926, 2016),
(3, 'F. Scott Fitzgerald', 'American', 1896, 1940),
(4, 'Aldous Huxley', 'British', 1894, 1963),
(5, 'J.D. Salinger', 'American', 1919, 2010),
(6, 'Herman Melville', 'American', 1819, 1891),
(7, 'Jane Austen', 'British', 1775, 1817),
(8, 'Leo Tolstoy', 'Russian', 1828, 1910),
(9, 'Fyodor Dostoevsky', 'Russian', 1821, 1881),
(10, 'J.R.R. Tolkien', 'British', 1892, 1973);
```
Books (as currently inserted — note the `id = 1` / `id = 7` issue mentioned above):
```sql
INSERT INTO books (id, title, author_id, genres, published_year, available_copies) VALUES
(1, 'Pride and Prejudice', 1, ARRAY['Romantic Novel'], 1813, ARRAY['Copy 1', 'Copy 2']),
(2, 'To Kill a Mockingbird', 2, ARRAY['Southern Gothic', 'Bildungsroman'], 1960, ARRAY['Copy 1', 'Copy 7']),
(3, 'The Great Gatsby', 3, ARRAY['Tragedy'], 1925, ARRAY['Copy 1', 'Copy 2']),
(4, 'Brave New World', 4, ARRAY['Dystopian', 'Science Fiction'], 1932, ARRAY['Copy 1', 'Copy 2']),
(5, 'The Catcher in the Rye', 5, ARRAY['Realist Novel', 'Bildungsroman'], 1951, ARRAY['Copy 1', 'Copy 2']),
(6, 'Moby-Dick', 6, ARRAY['Adventure Fiction'], 1851, ARRAY['Copy 1', 'Copy 2']),
(7, 'Pride and Prejudice', 7, ARRAY['Romantic Novel'], 1813, ARRAY['Copy 1', 'Copy 2']),
(8, 'War and Peace', 8, ARRAY['Historical Novel'], 1869, ARRAY['Copy 1', 'Copy 2']),
(9, 'Crime and Punishment', 9, ARRAY['Philosophical Novel'], 1866, ARRAY['Copy 1', 'Copy 2']),
(10, 'The Hobbit', 10, ARRAY['Fantasy'], 1937, ARRAY['Copy 1', 'Copy 2']);
```
> **To fix the id = 1 mismatch** so it matches the original brief (`'1984'` by Orwell instead of a duplicate Pride and Prejudice):
> ```sql
> UPDATE books
> SET title = '1984', genres = ARRAY['Dystopian', 'Political Fiction'], published_year = 1949
> WHERE id = 1;
> ```
Patrons:
```sql
INSERT INTO patrons (id, name, email, borrowed_books) VALUES
(1, 'Alice Johnson', 'alice@example.com', ARRAY[]::INT[]),
(2, 'Bob Smith', 'bob@example.com', ARRAY[1, 2]),
(3, 'Carol White', 'carol@example.com', ARRAY[]::INT[]),
(4, 'David Brown', 'david@example.com', ARRAY[3]),
(5, 'Eve Davis', 'eve@example.com', ARRAY[]::INT[]),
(6, 'Frank Moore', 'frank@example.com', ARRAY[4, 5]),
(7, 'Grace Miller', 'grace@example.com', ARRAY[]::INT[]),
(8, 'Hank Wilson', 'hank@example.com', ARRAY[6]),
(9, 'Ivy Taylor', 'ivy@example.com', ARRAY[]::INT[]),
(10, 'Jack Anderson', 'jack@example.com', ARRAY[7, 8]);
```
---
Sprint 3: Read Operations (Queries)
Get all books:
```sql
SELECT * FROM books;
```
Get a book by title:
```sql
SELECT * FROM books WHERE title = 'The Hobbit';
```
Get all books by a specific author:
```sql
SELECT b.*
FROM books b
JOIN author a ON b.author_id = a.id
WHERE a.name = 'George Orwell';
```
Get all available books (i.e. `available_copies` is not empty):
```sql
SELECT * FROM books
WHERE cardinality(available_copies) > 0;
```
---
Sprint 4: Update Operations
Mark a book as borrowed — remove one copy from `available_copies`:
```sql
BEGIN;

UPDATE books
SET available_copies = array_remove(available_copies, 'Copy 1')
WHERE id = 1
  AND cardinality(available_copies) > 0;

UPDATE patrons
SET borrowed_books = array_append(borrowed_books, 1)
WHERE id = 1;

COMMIT;
```
> This is the corrected version of the transaction from earlier — the original used `available_copies = available_copies - 1` and `WHERE books_id = 1`, which failed because `available_copies` is a `TEXT[]` (you can't subtract an integer from an array) and the column is `id`, not `books_id`. `array_remove` takes out a specific copy label instead.
A "return" (reverse of the above) — add a copy back:
```sql
UPDATE books
SET available_copies = array_append(available_copies, 'Copy 1')
WHERE id = 1;

UPDATE patrons
SET borrowed_books = array_remove(borrowed_books, 1)
WHERE id = 1;
```
Add a new genre to an existing book:
```sql
UPDATE books
SET genres = array_append(genres, 'Classic')
WHERE id = 1;
```
Add a borrowed book to a patron's record:
```sql
UPDATE patrons
SET borrowed_books = array_append(borrowed_books, 1)
WHERE id = 1;
```
---
Sprint 5: Delete Operations
Delete a book by title:
```sql
DELETE FROM books WHERE title = 'The Hobbit';
```
Delete an author by ID:
```sql
DELETE FROM author WHERE id = 10;
```
> Because `books.author_id` references `author.id`, this will fail while `id = 10` still has books linked to it — delete or reassign those books first, or add `ON DELETE CASCADE` to the foreign key if you want deletes to cascade automatically.
---
Sprint 6: Advanced Queries
Find books published after 1950:
```sql
SELECT * FROM books WHERE published_year > 1950;
```
Find all American authors:
```sql
SELECT * FROM author WHERE nationality = 'American';
```
Set all books as available (give every book back its two standard copies):
```sql
UPDATE books
SET available_copies = ARRAY['Copy 1', 'Copy 2'];
```
Find all books that are available AND published after 1950:
```sql
SELECT * FROM books
WHERE cardinality(available_copies) > 0
  AND published_year > 1950;
```
Find authors whose names contain "George":
```sql
SELECT * FROM author WHERE name LIKE '%George%';
```
Increment the published year 1869 by 1:
```sql
UPDATE books
SET published_year = published_year + 1
WHERE published_year = 1869;
```
---
Running the Queries
Option A: pgAdmin
Connect to your PostgreSQL server in pgAdmin.
Right-click Databases → Create → Database, name it `LibraryDB`.
Right-click `LibraryDB` → Query Tool.
Paste the SQL from each section and run with ▶ or `F5`.
Option B: psql
```bash
psql -U postgres
```
```sql
CREATE DATABASE LibraryDB;
\c LibraryDB
```
Or run a saved script directly:
```bash
psql -U postgres -d LibraryDB -f schema.sql
```
---
Schema Summary
Table	Key Columns
`author`	`id` (PK), `name`, `nationality`, `birth_year`, `death_year`
`books`	`id` (PK), `title`, `author_id` (FK → `author.id`), `genres` (array), `published_year`, `available_copies` (array)
`patrons`	`id` (PK), `name`, `email`, `book_id` (unused), `borrowed_books` (array of book ids)
