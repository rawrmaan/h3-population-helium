# H3 Population Helium

An global population dataset based on Uber's H3 mapping system and seeded from the open source [Kontur Population](https://data.humdata.org/dataset/kontur-population-dataset) dataset.

Optimized for easy consumption in [Helium](https://www.helium.com/) mapping projects.

## Using the data

Download the latest `kontur_population_20211109.csv` from this repository. The CSV file contains two columns:
- **hex_res8:** h3 index truncated to first 10 chars
- **population:** total human population in this hex

## Preparing Kontur Population dataset in PostgreSQL from scratch

Although the Kontur Population dataset uses H3 resolution 8 to segment population data, it doesn't  provide the H3 index of each hex. Instead, it provides a set of points that make up each hex.

We need the H3 index of each hex to easily join and compare with data in the Helium ecosystem. Let's calculate the H3 indices ourselves!

### Requirements

- PostgreSQL 9 or higher with PostGIS extension installed
- ogr2ogr CLI (instructions below)

### 1. Download and extract

Download the latest dataset from [this page](https://data.humdata.org/dataset/kontur-population-dataset).

The file will be of the format `.gpkg.gz`. Extract the `.gpkg` file to your working directory.

### 2. Use ogr2ogr to import gpkg into Postgres

You'll need `ogr2ogr` CLI installed for this.

If you're using Homebrew on macOS, the following command will accomplish this:

```bash
brew install gdal
```

Then, run the following command to import into your Postgres DB. This assumes you're running Postgres on `localhost` and that your DB name is `heliumpop`.

```bash
ogr2ogr -f PostgreSQL "PG:dbname=heliumpop" kontur_population_20211109.gpkg
```

This will import the data into a table called `kontur_population`. NOTE: If the import is stopped partway through, make sure to `TRUNCATE` this table before restarting the import or you will get duplicate data.

### 3. Convert geometry to SRID 4326 and find centers

The dataset includes hexagons in the format of PostGIS `Geometry` objects with SRID 3857 (Spherical Mercator). Since we're looking to get the H3 index of each hex, we need to convert the hex geometry to lat/long (SRID 4326) and find the center.

Open `psql` or your favorite Postgres client and run the following queries:

```sql
alter table kontur_population
add column center geometry(Geometry,4326);

update kontur_population
set center = st_centroid(st_transform(geom,4326));
```

### 4. Add H3 index to dataset

We're almost there! Add a new column for the H3 index of each hex and create a unique index to ensure consistency.

```sql
alter table kontur_population
add column hex_res8 text;

create unique index on kontur_population (hex_res8);
```

Install required dependencies for the next step:

```bash
npm install
```

Copy `.env.sample` to a new file called `.env` and add your Postgres database credentials. Then, run the H3 index update script. This will add H3 indexes, truncated to the first 10 significant characters, to the `hex_res8` column of the `kontur_population` table:

```bash
npm run update-h3-index
```

And that's it! You can create the final using the following query and export to CSV using your favorite tool:

```sql
select hex_res8, population
from kontur_population
```
