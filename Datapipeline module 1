# -data_pipeline-analytics-support_assistant-
Capstone project
#writefile scrape_pipeline.py

from __future__ import annotations

import re
import sqlite3
from pathlib import Path
from typing import Optional

import pandas as pd
import requests
from bs4 import BeautifulSoup

BASE_URL = "https://books.toscrape.com/"
RATE_GBP_TO_INR = 105.50
PAGES_TO_SCRAPE = 5

# Corrected paths for Colab environment
DB_PATH = Path("/content/books_catalog.sqlite")
OUTPUT_DIR = Path("/content/outputs")

RATING_MAP = {"One": 1, "Two": 2, "Three": 3, "Four": 4, "Five": 5}


def get_soup(session: requests.Session, url: str) -> BeautifulSoup:
    response = session.get(url, timeout=20)
    response.raise_for_status()
    return BeautifulSoup(response.text, "html.parser")


def parse_price(text: str) -> Optional[float]:
    try:
        return float(re.sub(r"[^\d.]", "", text))
    except (TypeError, ValueError):
        return None


def parse_rating(text: str) -> Optional[int]:
    return RATING_MAP.get(str(text).strip())


def parse_stock(text: str) -> Optional[bool]:
    value = str(text).strip().lower()
    if "in stock" in value:
        return True
    if "out of stock" in value:
        return False
    return None


def scrape_books() -> pd.DataFrame:
    """Scrape the first five All Products pages (20 books/page = at least 60)."""
    rows = []

    with requests.Session() as session:
        session.headers.update({"User-Agent": "Mozilla/5.0 data-pipeline-course"})
        for page in range(1, PAGES_TO_SCRAPE + 1):
            listing_url = BASE_URL if page == 1 else f"{BASE_URL}catalogue/page-{page}.html"
            soup = get_soup(session, listing_url)

            for article in soup.select("article.product_pod"):
                link = article.select_one("h3 a")
                if not link:
                    continue

                detail_url = requests.compat.urljoin(listing_url, link.get("href", ""))
                detail = get_soup(session, detail_url)

                category = None
                breadcrumb = detail.select("ul.breadcrumb li a")
                if breadcrumb:
                    category = breadcrumb[-1].get_text(strip=True)

                price_node = detail.select_one("p.price_color")
                rating_node = detail.select_one("p.star-rating")
                availability_node = detail.select_one("p.availability")

                rating_text = None
                if rating_node:
                    classes = rating_node.get("class", [])
                    rating_text = next((c for c in classes if c in RATING_MAP), None)

                rows.append(
                    {
                        "title": link.get("title") or link.get_text(strip=True),
                        "price_gbp_raw": price_node.get_text(" ", strip=True) if price_node else None,
                        "star_rating_raw": rating_text,
                        "availability_raw": availability_node.get_text(" ", strip=True)
                        if availability_node else None,
                        "category": category,
                        "source_url": detail_url,
                    }
                )

    return pd.DataFrame(rows)


def clean_books(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()

    df["price_gbp"] = df["price_gbp_raw"].apply(parse_price)
    df["rating"] = df["star_rating_raw"].apply(parse_rating)
    df["in_stock"] = df["availability_raw"].apply(parse_stock)

    # Numeric parse failures use median imputation, as required by the assignment.
    for column in ["price_gbp", "rating"]:
        median = df[column].median()
        if pd.isna(median):
            raise ValueError(f"No usable numeric values remain for {column}")
        df[column] = df[column].fillna(median)

    # Availability/category are non-numeric and cannot be median-imputed.
    # Unexpected/missing values are dropped so the relational dataset stays valid.
    before = len(df)
    df = df.dropna(subset=["in_stock", "category", "title"]).copy()
    dropped = before - len(df)
    print(f"Dropped {dropped} rows with missing/unparseable stock/category/title.")

    df["rating"] = df["rating"].round().astype(int).clip(1, 5)
    df["in_stock"] = df["in_stock"].astype(bool)
    df["price_inr"] = (df["price_gbp"] * RATE_GBP_TO_INR).round(2)

    # Keep the cleaned analytical columns plus source URL for traceability.
    columns = [
        "title", "price_gbp", "price_inr", "rating",
        "in_stock", "category", "source_url"
    ]
    df = df[columns].drop_duplicates(subset=["title", "category"]).reset_index(drop=True)

    if len(df) < 60:
        raise ValueError(f"Only {len(df)} clean rows produced; at least 60 are required.")
    if df["category"].nunique() < 3:
        raise ValueError("Fewer than 3 categories were found.")

    return df


def create_database(df: pd.DataFrame, db_path: Path = DB_PATH) -> None:
    if db_path.exists():
        db_path.unlink()

    conn = sqlite3.connect(db_path)
    try:
        conn.execute("PRAGMA foreign_keys = ON")

        conn.executescript(
            """
            CREATE TABLE categories (
                category_id INTEGER PRIMARY KEY,
                category_name TEXT NOT NULL UNIQUE
            );

            CREATE TABLE books (
                book_id INTEGER PRIMARY KEY,
                title TEXT NOT NULL,
                price_gbp REAL NOT NULL,
                price_inr REAL NOT NULL,
                rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
                in_stock INTEGER NOT NULL CHECK (in_stock IN (0, 1)),
                category_id INTEGER NOT NULL,
                source_url TEXT,
                FOREIGN KEY (category_id) REFERENCES categories(category_id)
            );
            """
        )

        categories = sorted(df["category"].unique())
        conn.executemany(
            "INSERT INTO categories(category_name) VALUES (?)",
            [(c,) for c in categories],
        )

        category_map = dict(
            conn.execute("SELECT category_name, category_id FROM categories").fetchall()
        )

        book_rows = [
            (
                row.title,
                float(row.price_gbp),
                float(row.price_inr),
                int(row.rating),
                int(row.in_stock),
                category_map[row.category],
                row.source_url,
            )
            for row in df.itertuples(index=False)
        ]

        conn.executemany(
            """
            INSERT INTO books
            (title, price_gbp, price_inr, rating, in_stock, category_id, source_url)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            book_rows,
        )
        conn.commit()
    finally:
        conn.close()


QUERIES = {
    "01_select_where": """
        SELECT title, price_gbp, rating
        FROM books
        WHERE in_stock = 1 AND price_gbp < 20
        ORDER BY price_gbp;
    """,
    "02_order_by": """
        SELECT title, price_gbp, price_inr
        FROM books
        ORDER BY price_inr DESC;
    """,
    "03_limit": """
        SELECT title, rating, price_gbp
        FROM books
        ORDER BY rating DESC, price_gbp ASC
        LIMIT 10;
    """,
    "04_distinct": """
        SELECT DISTINCT category_name
        FROM categories
        ORDER BY category_name;
    """,
    "05_between": """
        SELECT title, price_gbp, rating
        FROM books
        WHERE price_gbp BETWEEN 10 AND 20
        ORDER BY price_gbp;
    """,
    "06_join": """
        SELECT
            c.category_name,
            b.title,
            b.rating,
            b.price_gbp,
            b.price_inr,
            b.in_stock
        FROM books AS b
        JOIN categories AS c
          ON b.category_id = c.category_id
        ORDER BY c.category_name, b.rating DESC, b.price_gbp ASC;
    """,
}


def run_queries(db_path: Path = DB_PATH) -> dict[str, pd.DataFrame]:
    OUTPUT_DIR.mkdir(exist_ok=True)
    conn = sqlite3.connect(db_path)
    try:
        results = {}
        with open(OUTPUT_DIR / "queries.sql", "w", encoding="utf-8") as sql_file, \
             open(OUTPUT_DIR / "query_outputs.txt", "w", encoding="utf-8") as out_file:

            for name, query in QUERIES.items():
                sql_file.write(f"-- {name}\n{query.strip()}\n\n")
                result = pd.read_sql(query, conn)
                results[name] = result

                out_file.write(f"\n{'=' * 80}\n{name}\n{'=' * 80}\n")
                out_file.write(result.to_string(index=False))
                out_file.write("\n")

                print(f"\n{name}\n{result.to_string(index=False)}")

        return results
    finally:
        conn.close()


def demonstrate_pandas_equivalence(df: pd.DataFrame, db_path: Path = DB_PATH) -> None:
    """Compare SQL JOIN output with pd.merge() on the in-memory DataFrames."""
    conn = sqlite3.connect(db_path)
    try:
        sql_join = pd.read_sql(QUERIES["06_join"], conn)
        categories_df = pd.read_sql(
            "SELECT category_id, category_name FROM categories ORDER BY category_id",
            conn,
        )
        books_df = pd.read_sql(
            """
            SELECT book_id, title, rating, price_gbp, price_inr, in_stock, category_id
            FROM books
            """,
            conn,
        )
    finally:
        conn.close()

    merged = books_df.merge(categories_df, on="category_id", how="inner")
    merged = merged[
        ["category_name", "title", "rating", "price_gbp", "price_inr", "in_stock"]
    ].sort_values(
        ["category_name", "rating", "price_gbp"],
        ascending=[True, False, True],
    ).reset_index(drop=True)

    sql_normalized = sql_join.copy()
    sql_normalized["in_stock"] = sql_normalized["in_stock"].astype(int)
    merged["in_stock"] = merged["in_stock"].astype(int)

    equivalent = sql_normalized.equals(merged)

    comparison_path = OUTPUT_DIR / "join_comparison.txt"
    with open(comparison_path, "w", encoding="utf-8") as f:
        f.write("SQL pd.read_sql JOIN result:\n")
        f.write(sql_normalized.to_string(index=False))
        f.write("\n\npd.merge() result:\n")
        f.write(merged.to_string(index=False))
        f.write(f"\n\nEquivalent: {equivalent}\n")

    print("\nJOIN equivalence check:", equivalent)
    print("\nSQL JOIN result:\n", sql_normalized.to_string(index=False))
    print("\npd.merge() result:\n", merged.to_string(index=False))

    if not equivalent:
        raise AssertionError("SQL JOIN and pd.merge() results are not equivalent.")


def main() -> None:
    OUTPUT_DIR.mkdir(exist_ok=True)

    raw = scrape_books()
    raw.to_csv(OUTPUT_DIR / "raw_books.csv", index=False)

    clean = clean_books(raw)
    clean.to_csv(OUTPUT_DIR / "clean_books.csv", index=False)

    print(f"\nClean rows: {len(clean)}")
    print(f"Categories: {clean['category'].nunique()}")
    print(f"Fixed conversion rate: 1 GBP = {RATE_GBP_TO_INR:.2f} INR")

    create_database(clean)
    print(f"SQLite database created: {DB_PATH}")

    run_queries()
    demonstrate_pandas_equivalence(clean)


if __name__ == "__main__":
    main()
