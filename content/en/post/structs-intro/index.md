---
title: "📚 SQL Structs: When Your Database Learns to Think Like a Bookshelf"
date: 2025-06-17
tags: ["sql", "structs", "data-modeling", "bigquery", "spark", "database-design", "normalization", "denormalization", "arrays", "nested-data"]
categories: ["Data Engineering"]
---

Imagine your database as a giant library. For decades, we've been organizing it like a traditional library catalog system – every book detail gets its own card, filed in separate drawers. Title cards in one drawer, author cards in another, publication year in yet another. This is what we call **normalization** in database terms.

But what if your library could store complete book information in a single, smart envelope? That envelope could contain the title, author, publication details, and even reviews – all tucked neatly together. This is essentially what a **struct** does in modern SQL databases.

## What Exactly is a Struct?

A struct is like a Swiss Army knife for your data. Instead of forcing you to store related information in separate tables (like our library cards), a struct lets you bundle related fields together into one logical unit.

```sql
-- Traditional approach: Multiple tables
-- books: book_id, title
-- authors: book_id, author_name
-- publishers: book_id, publisher_name, year

-- Struct approach: One table with bundled data
CREATE TABLE books (
  book_id INT64,
  info STRUCT<
    title STRING,
    author STRING,
    publisher STRING,
    year INT64
  >
)
```

Think of it like the difference between a filing cabinet (traditional tables) and a well-organized briefcase (structs). Both can store your documents, but the briefcase keeps related items together in logical compartments.

## The Magic of Arrays of Structs

Now, here's where things get really interesting. What if our book envelope could contain multiple books? That's what an **array of structs** does – it's like having a expandable folder that can hold multiple complete book records.

```sql
-- A customer's reading history
CREATE TABLE customers (
  customer_id INT64,
  name STRING,
  reading_history ARRAY<STRUCT<
    book_title STRING,
    genre STRING,
    rating INT64,
    date_read DATE
  >>
)
```

Imagine a customer named Sarah whose reading history looks like this:
```
Sarah's Reading History:
📚 [
  {title: "The Hobbit", genre: "Fantasy", rating: 5, date_read: "2025-01-15"},
  {title: "Dune", genre: "Sci-Fi", rating: 4, date_read: "2025-02-20"},
  {title: "Pride and Prejudice", genre: "Romance", rating: 5, date_read: "2025-03-10"}
]
```

All of Sarah's reading history is stored in one place, like a personal reading journal that travels with her customer record.

## The Great Database Philosophy Debate

This brings us to one of the biggest debates in data engineering: **Should we organize our data like a library catalog (normalized) or like personal filing cabinets (denormalized with structs)?**

### The Library Catalog Approach (Normalization)
- **Pros**: No duplicate information, easy to update one piece of data
- **Cons**: Need to visit multiple "drawers" (tables) to get complete information
- **Best for**: When you frequently update individual pieces of information

### The Personal Filing Cabinet Approach (Structs)
- **Pros**: Everything related is in one place, faster to retrieve complete information
- **Cons**: Updating one piece means reorganizing the whole folder
- **Best for**: When you mostly read data and rarely update individual pieces

## Why Structs Struggle with Frequent Updates

Let's use a restaurant analogy. Imagine you're a waiter taking orders:

**Traditional Tables (Normalized)**: Each order item is written on a separate ticket. If a customer changes their mind about one item, you just throw away that ticket and write a new one.

**Structs**: The entire order is written on one big ticket. If the customer changes their mind about one item, you have to erase the entire ticket and rewrite everything from scratch.

```sql
-- Easy update with normalized tables
UPDATE order_items SET quantity = 3 WHERE order_id = 123 AND item = 'Pizza';

-- Complex update with structs - you have to rewrite the entire array
UPDATE orders 
SET items = ARRAY(
  SELECT CASE 
    WHEN item.name = 'Pizza' 
    THEN STRUCT(item.name, 3 as quantity, item.price)
    ELSE item
  END
  FROM UNNEST(items) as item
)
WHERE order_id = 123;
```

This is why structs work great for analytics (where you mostly read data) but can be problematic for transactional systems (where you frequently update individual pieces).

## The Data Engineer's Survival Guide to Structs

### 1. **Schema Evolution: Growing Pains**
Structs are like teenagers – they grow and change, but not always gracefully. Adding new fields is usually fine, but removing or renaming fields can break everything downstream.

### 2. **Storage Considerations: Size Matters**
Think of structs like packing a suitcase. A well-organized struct is like efficient packing – everything fits nicely. But an overstuffed struct with lots of empty fields is like packing a suitcase with mostly air – wasteful and inefficient.

### 3. **Performance Patterns: The Highway vs. Side Streets**
- **Good**: Retrieving complete related data (like taking the highway)
- **Bad**: Searching for specific nested values (like navigating side streets)

### 4. **Cross-Platform Compatibility: Speaking Different Languages**
Different database systems handle structs like different countries handle currency. BigQuery loves them, Spark speaks their language fluently, but older systems might need a translator (or might not understand them at all).

### 5. **Memory Usage: The Multiplication Effect**
When you "explode" an array of structs in memory, it's like unpacking a Russian nesting doll – what started as one compact object suddenly becomes many individual pieces, each taking up memory space.

## Common Struct Pitfalls (And How to Avoid Them)

### The "Inception" Problem
Don't go too deep with nested structs. It's like trying to find a specific book inside a box, inside another box, inside a filing cabinet, inside a storage unit. Eventually, nobody remembers where anything is.

### The "Everything Bagel" Syndrome
Just because you *can* put everything in one struct doesn't mean you *should*. It's like having one junk drawer that contains everything from batteries to birthday candles – technically efficient, but practically a nightmare.

### The "Update Nightmare"
Using structs for frequently changing data is like using a laminated restaurant menu – every small change requires reprinting the whole thing.

## When to Use Structs: The Decision Framework

**Choose Structs When:**
- You have naturally nested data (like addresses, product catalogs, or user profiles)
- You mostly read data and rarely update individual pieces
- You need to query relationships between nested fields
- You're building analytics or reporting systems

**Avoid Structs When:**
- You frequently update individual pieces of data
- You need to search or index specific nested values regularly
- You're building transactional systems with high concurrency
- Your team isn't comfortable with the added complexity

## The Bottom Line

Structs are like having a smart personal assistant for your data – they can organize related information beautifully and make certain tasks much easier. But like any assistant, they need to be used appropriately. They're not a replacement for good old-fashioned database design principles; they're an additional tool in your toolkit.

The key is understanding when to use a filing cabinet (traditional tables) versus when to use a well-organized briefcase (structs). Both have their place in a data engineer's world, and the best solutions often use both approaches strategically.

> Remember: structs represent a shift from thinking in "tables and rows" to thinking in "documents and objects." Embrace this paradigm shift, but don't abandon the lessons learned from decades of relational database design. The future of data engineering lies not in choosing one approach over another, but in knowing when to use each tool effectively.

