# AWS Spotify Data Pipeline Workshop

**Services:** S3 · Glue · Athena · QuickSight · IAM  
**Region:** US-East-1 (N. Virginia) — use this for every step  
**Dataset:** [Spotify Dataset 2023 — Kaggle](https://www.kaggle.com/datasets/tonygordonjr/spotify-dataset-2023?resource=download&select=spotify_features_data_2023.csv)  
**Slides:** [Workshop Slides — Google Slides](https://docs.google.com/presentation/d/12LN-QsTvhKxYfG1BVBkGtytWirWhs4qY2sFHljJjWQs/edit?usp=sharing)

---

## 1. Account Setup

- Create an AWS account
- Sign up for QuickSight
  - During signup, grant access to **Athena** and **S3**

---

## 2. Create IAM User

- Go to IAM → Users → Create user
- Attach the following policies:

| Policy | Type |
|--------|------|
| `AWSGlueServiceRole` | AWS managed |
| `CloudWatchLogsReadOnlyAccess` | AWS managed |
| `AmazonS3FullAccess` | AWS managed |
| `AWSGlueConsoleFullAccess` | AWS managed |
| `AmazonAthenaFullAccess` | AWS managed |
| `AWSQuickSightAthenaAccess` | AWS managed |
| `AWSQuickSightDescribeRDS` | AWS managed |
| `iampasson` | Inline policy (below) |

- Create an inline policy named `iampasson` with the following JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::944150635171:role/Glue_access_s3"
    }
  ]
}
```

---

## 3. Create Glue IAM Role

- Go to IAM → Roles → Create role
  - **Trusted entity type:** AWS Service
  - **Use case:** Glue
- Attach the following policies:

| Policy |
|--------|
| `AmazonS3FullAccess` |
| `AWSGlueServiceRole` |
| `AWSGlueConsoleFullAccess` |
| `AmazonAthenaFullAccess` |
| `AWSQuickSightAthenaAccess` |

- Name the role `Glue_access_s3`

> Sign in as the IAM user and change your password before proceeding.

---

## 4. Create S3 Bucket

- Create a bucket with default settings
- Inside the bucket create the following folders:
  - `staging/`
  - `datawarehouse/`
  - `athena/`
- Download the Spotify 2023 dataset from Kaggle:  
  https://www.kaggle.com/datasets/tonygordonjr/spotify-dataset-2023?resource=download
- Upload the three CSV files into `staging/`:
  - `spotify_artist_data_2023.csv`
  - `spotify_tracks_data_2023.csv`
  - `spotify-albums_data_2023.csv`

---

## 5. Glue: Run Visual ETL Job

- Go to Glue → ETL Jobs → Visual ETL → create a new blank canvas
- Add three **S3 DataSource** nodes pointing to each file in `staging/`
- **ApplyMapping** on Albums — rename `track_id → alb_track_id` to prevent column collision
- **ApplyMapping** on Tracks — rename `id → track_id`
- **Join 1** — Albums + Artists
  - Type: Left join
  - Condition: `Albums.artist_id = Artists.id`
- **Join 2** — (Albums+Artists) + Tracks
  - Type: Left join
  - Condition: `alb_track_id = track_id`
- **Drop Fields** — remove redundant columns, preview the output
- **S3 Target** — set destination to `datawarehouse/`, format Parquet, compression Snappy
- Under **Job details** — set IAM role to `Glue_access_s3`
- Save and run the job

---

## 6. Glue: Create Crawler

- Go to Glue → Databases → create a new database (e.g. `spotify`)
- Go to Glue → Crawlers → Create crawler
  - Set S3 data source to your `datawarehouse/` path
  - Set output database to the database created above
  - Set IAM role to `Glue_access_s3`
- Run the crawler
- Verify the table `datawarehouse` appears under the `spotify` database

---

## 7. Athena: Verify the Table

- Go to Athena → Query editor
- Set workgroup query results location to `s3://your-bucket/athena/`
- Run the following query to confirm the schema:

```sql
DESCRIBE spotify.datawarehouse;
```

- Optionally run a data check:

```sql
SELECT * FROM spotify.datawarehouse LIMIT 10;
```

---

## 8. QuickSight: Build Dashboard

### Create dataset
- Go to QuickSight → Datasets → New dataset → Athena
- Select workgroup `primary` → database `spotify` → table `datawarehouse`
- Use custom SQL to cast numeric columns and limit rows:

```sql
SELECT
    name,
    CAST(artist_popularity AS INTEGER)            AS artist_popularity,
    CAST(followers AS INTEGER)                    AS followers,
    CAST(track_alb_track_popularity AS INTEGER)   AS track_alb_track_popularity,
    CAST(track_alb_duration_sec AS DOUBLE)        AS track_alb_duration_sec,
    genre_0,
    track_alb_track_name,
    track_alb_album_name,
    track_alb_release_date,
    track_alb_label,
    track_alb_explicit,
    track_alb_album_type,
    track_alb_total_tracks
FROM spotify.datawarehouse
LIMIT 1000
```

- Select **Directly query your data** (not SPICE)
- Click **Visualize**

### Create visualizations
- **Top 20 artists by popularity** — vertical bar chart
  - X axis: `name`
  - Value: `artist_popularity` (Max)
  - Sort by `artist_popularity` descending
  - Visual menu → Top/bottom ranked → Top 20
- **Top 20 artists by followers** — vertical bar chart
- **Track popularity distribution** — histogram on `track_alb_track_popularity`
- **Explicit vs non-explicit tracks** — pie chart on `track_alb_explicit`
- **Releases over time** — line chart on `track_alb_release_date`
- **Top labels by track count** — bar chart on `track_alb_label` (Count)

### Publish
- Click **Publish** → name your dashboard → confirm
