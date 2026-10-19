# silvershop/category-index

A derived, read-only flat index of **product → effective category** membership for
[SilverShop](https://github.com/silvershop/silvershop-core) (SilverStripe CMS 6), so category filtering is a single
indexed JOIN instead of an `OR` across two relations plus a per-level tree walk.

## The problem

A SilverShop product's **primary** category is its page-tree parent (`ParentID`); **secondary** categories live in
the editable `ProductCategories` many_many. So there is no single relation you can query/index for "all products in
category X". Core merges the two at runtime:

- `Product::getCategoryIDs()` / `getCategories()` — ancestors (`ParentID` chain) + `ProductCategories`.
- `ProductCategory::ProductsShowable()` — `filterAny(['ParentID' => …, 'ProductCategories.ID' => …])` plus, for the
  recursive default, `AllChildCategoryIDs()` (one query per tree level).

That's fine for in-app rendering, but you can't hand it to a search engine / a report / a raw SQL query, and on large
catalogs or deep trees the recursive listing does more work than it needs to.

## What this adds

- A flat, read-only many_many **`Product.EffectiveCategories`** holding the product's primary category, all its
  ancestor categories, and its secondary `ProductCategories`. Because it contains the **ancestors**, filtering on a
  category's own ID already includes every descendant — no tree walk.
- The editable `ProductCategories` stays the single source of truth; this relation is derived and disposable.

```php
// all products in this category (incl. descendants), one indexed JOIN:
$products = Product::get()->filter(['EffectiveCategories.ID' => $category->ID]);
```

With `optimise_products_showable` on (default), `ProductCategory::ProductsShowable()` is transparently routed through
the index for recursive listings, so the storefront gets the speed-up for free. The real win comes from core's
`overrideProductsShowable` extension hook (silvershop/core ^6): the index supplies the whole list up front, so core
skips the `AllChildCategoryIDs()` tree walk entirely. On a core without that hook the module falls back to
`updateProductsShowable` (correct, but runs after the walk).

## Requirements

- `silvershop/core` ^6
- `silverstripe/framework` ^6, `silverstripe/reports` ^6 / PHP 8.3+

## Installation

```bash
composer require silvershop/category-index
```

Then run a dev/build (`/dev/build?flush=1`) to add the `EffectiveCategories` join table, and backfill the index once:

```
/dev/tasks/SilverShop-CategoryIndex-RebuildCategoryIndexTask
```

The extensions are applied automatically to `Product` and `ProductCategory`; there is nothing to edit in the CMS (the
relation is derived and hidden).

## Maintenance

The index reflects the **published (Live) stage**, because the storefront reads Live — draft-only edits don't touch
it.

- **Product publish** → that product's index is rebuilt from Live (cheap). **Unpublish** → cleared.
- **Category publish** → the subtree is reindexed from Live (a move changes every descendant's ancestor set). This
  runs **inline**; on a large catalog it can touch thousands of products, so install
  `symbiote/silverstripe-queuedjobs` and push `CategoryStructureExtension::reindexSubtree()` to a job (see the note
  in that class). It currently fires on any category publish and could be narrowed to structural (`ParentID`) changes.
- **Bulk import / seed** → set `ProductCategoryIndexExtension::$disable_auto_reindex = true` around the import to skip
  per-write reindexing, then run the rebuild task once.
- **Backfill / repair** → `dev/tasks/SilverShop-CategoryIndex-RebuildCategoryIndexTask` (idempotent).

## Config

```yaml
SilverShop\CategoryIndex\CategoryStructureExtension:
  optimise_products_showable: true   # route ProductsShowable() through the index (recursive listings only)
```

The `ProductsShowable()` swap only applies when `ProductCategory.include_child_groups` is on (the recursive model the
index matches). Turn `optimise_products_showable` off to keep core's query while still using the index directly.

## Reports

Ships a **Sales by category** report (CMS → Reports): revenue and units sold per category for a period, rolled up
through the index so each category total includes its whole subtree (category rows therefore overlap). Requires
`silverstripe/reports`.

## Dev / benchmark tasks

Only meaningful in a dev environment:

- `SilverShop-CategoryIndex-CheckCategoryIndexTask` — verifies the index matches core's own recursive membership for
  every category and confirms `ProductsShowable()` is routed through the index. Exit code non-zero on drift.
- `SilverShop-CategoryIndex-Dev-SeedLargeCatalogTask` — builds/grows a deep "Benchmark Catalog" tree with products
  (draft) to measure on.
- `SilverShop-CategoryIndex-Dev-BenchmarkCategoryIndexTask` — times the recursive listing core vs index-override vs
  direct index query.
- `SilverShop-CategoryIndex-Dev-DeleteBenchmarkCatalogTask` — removes the benchmark catalog.

## License

BSD-3-Clause.

## Credits

Built and maintained by **Remy Vaartjes** (<remy@xd.nl>) — part of the **SilverShop-Suite**.
