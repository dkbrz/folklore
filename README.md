# Folklore

Set `FLASK_SECRET_KEY` to the same persistent, random secret on every app
worker. The app creates the `search_sessions` table at startup; the database
user needs permission to create it on first deployment. Search sessions expire
after one day of inactivity.

## MySQL search backend runbook

### 1) Apply the MySQL search schema

This repository ships the SQL schema for the MySQL search index. Apply it manually once per database:

```bash
mysql -u <user> -p <db> < migrations/mysql_search_index.sql
```

### 2) Install the required NLTK data

Sentence splitting relies on the NLTK `punkt` model:

```bash
python -m nltk.downloader punkt
```

### 3) Select a search backend

Set the backend via environment variable before starting the app:

```bash
export SEARCH_BACKEND=mysql
# or
export SEARCH_BACKEND=elasticsearch
```

### 4) Build or refresh the MySQL index

Use the Flask CLI to build the MySQL sentence index:

```bash
flask search-index rebuild --truncate --batch-size 1000
```

Rebuilds replace the index in one database transaction, so the old index stays
available if rebuilding fails. A large rebuild can hold database locks and needs
enough database space for the transaction. Edits made through the admin update
the MySQL index in the same transaction. The `--truncate` option remains
accepted for compatibility; every rebuild now replaces the full index.

To index a single text by id:

```bash
flask search-index text --id 123
```

### 5) MySQL FULLTEXT configuration notes

The sentence index uses a MySQL FULLTEXT index. MySQL may ignore short tokens or apply stopword filtering by default. If you need short tokens or specific languages, review and adjust the MySQL configuration options below, then rebuild the index:

- `innodb_ft_min_token_size`
- `ft_min_word_len`
- `innodb_ft_enable_stopword`
- `innodb_ft_server_stopword_table`

After changing any FULLTEXT settings, restart MySQL and rebuild the index with `flask search-index rebuild --truncate`.
