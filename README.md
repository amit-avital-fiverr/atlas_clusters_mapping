# MongoDB Atlas Metadata Collector

This repository contains Python scripts to collect and analyze cluster metadata from MongoDB Atlas.

## Scripts

### `atlas_metadata_collector.py`
Collects cluster metadata from MongoDB Atlas for all projects in an organization.

### `cluster_check.py`
Checks clusters in a specific MongoDB Atlas project with full metrics collection (CPU, Memory, IOPS, Disk, Connections, Operations).

## Features

- Collects metadata for all projects in an organization
- For each cluster, gathers:
  - Cluster name, ID, type
  - MongoDB version
  - Provider (AWS, Azure, GCP)
  - Region
  - Tier/Instance size
  - Disk size
  - State
  - Created date
  - Resource usage metrics (CPU, Memory, IOPS, Disk, Connections, Operations)

## Requirements

- Python 3.7+
- MongoDB Atlas API credentials

## Installation

```bash
pip3 install -r requirements.txt
```

## Usage

Both scripts support command-line arguments and environment variables for credentials.

### Environment Variables

You can set credentials using environment variables or a `.env` file:

```bash
export ATLAS_PUBLIC_KEY="your-public-key"
export ATLAS_PRIVATE_KEY="your-private-key"
export ATLAS_ORG_ID="your-org-id"
export ATLAS_PROJECT_ID="your-project-id"
```

### atlas_metadata_collector.py

Collects metadata from all projects in an organization. The script supports both JSON and CSV output formats and optional time-based filtering:

#### JSON Output

```bash
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --output atlas_metadata.json \
  --pretty
```

#### CSV Output

```bash
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --output atlas_metadata.csv
```

#### Advanced Filtering Options

**Time-Based Filtering** - Filter metrics to specific hours of the day (all times in UTC):

```bash
# Business hours only (2 PM to 11:59 PM UTC)
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --time-filter-start 14:00 \
  --time-filter-end 23:59 \
  --output business_hours.csv

# Night shift (10 PM to 6 AM UTC - cross-midnight range)
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --time-filter-start 22:00 \
  --time-filter-end 06:00 \
  --output night_shift.json
```

**Project and Cluster Filtering** - Focus on specific projects or clusters:

```bash
# Process only a specific project
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --project-filter "Production Environment" \
  --output production.csv

# Process only a specific cluster (requires project filter)
python3 atlas_metadata_collector.py \
  --org-id YOUR_ORG_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --project-filter "507f1f77bcf86cd799439012" \
  --cluster-filter "main-cluster" \
  --output single_cluster.json
```

The output format is automatically detected by the file extension (`.json` or `.csv`).

### cluster_check.py

Checks clusters in a specific project and outputs detailed metrics to `clusters_check.json`:

```bash
python3 cluster_check.py \
  --project-id YOUR_PROJECT_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY
```

Or using environment variables:

```bash
python3 cluster_check.py
```

#### Advanced Filtering for Single Project

**Time-Based Filtering** - Filter metrics to specific hours of the day (all times in UTC):

```bash
# Business hours analysis
python3 cluster_check.py \
  --project-id YOUR_PROJECT_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --time-filter-start 14:00 \
  --time-filter-end 23:59

# Night shift analysis (cross-midnight range)
python3 cluster_check.py \
  --project-id YOUR_PROJECT_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --time-filter-start 22:00 \
  --time-filter-end 06:00
```

**Cluster Filtering** - Focus on a specific cluster within the project:

```bash
# Analyze single cluster only
python3 cluster_check.py \
  --project-id YOUR_PROJECT_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --cluster-filter "main-production-cluster"

# Combine cluster and time filtering
python3 cluster_check.py \
  --project-id YOUR_PROJECT_ID \
  --public-key YOUR_PUBLIC_KEY \
  --private-key YOUR_PRIVATE_KEY \
  --cluster-filter "analytics-cluster" \
  --time-filter-start 09:00 \
  --time-filter-end 17:00
```

## Usage Flags Calculation

The scripts calculate low usage flags based on tier specifications loaded from `atlas_aws.csv`. The tier limits are matched by joining the cluster's `tier` field from the API data to the `tier` column in the CSV file.

### Flag Calculations

- **`low_iops_use`**: `true` if `iops_avg < 0.75 * iops_tier_limit`
  - Compares average IOPS usage against 75% of compare tier's IOPS limit
  
- **`low_memory_use`**: `true` if `memory_max_gb < memory_tier_limit_gb * 0.75`
  - Compares maximum memory usage against 75% of compare tier's RAM limit
  
- **`low_cpu_use`**: `true` if `cpu_avg_percent < 37` (standard tiers) or `cpu_avg_percent < 15` (M10/M20 burstable tiers)
  - Flags clusters with average CPU usage below threshold
  - M10/M20 tiers use burstable CPU with 20% baseline, so threshold is 75% of baseline = 15%
  - All other tiers use 37% threshold based on standard CPU performance
  
- **`cpu_burstable_lower_tier`**: `true` for M10/M20 clusters, `false` for all other tiers
  - Indicates whether the cluster uses MongoDB Atlas burstable CPU performance
  - Burstable tiers (M10/M20) have different CPU baseline calculations than dedicated tiers
  
- **`low_disk_use`**: `true` if `disk_usage_max_gb < disk_size_gb * 0.3`
  - Flags clusters using less than 30% of their allocated disk space

### Tier Specifications

Tier limits (CPU, RAM, IOPS) are loaded from `atlas_aws.csv`, which contains tier specifications. The CSV must have columns: `tier`, `cpu`, `ram`, `connection`, and `iops`. Clusters with tiers not found in the CSV will have `null` values for tier limits and usage flags.

**Burstable CPU Tiers**: M10 and M20 clusters use MongoDB Atlas burstable CPU performance with a 20% baseline. The scripts automatically detect these tiers and apply the appropriate CPU threshold (15% instead of 37%) for low usage calculations.

## Time-Based Metrics Filtering

Both scripts now support filtering metrics collection to specific hour ranges within each day:

### Features
- **UTC Timestamps**: All filtering uses UTC time (Atlas API native format)
- **Cross-Midnight Support**: Ranges like `22:00-06:00` work natively for night shifts
- **Warning Messages**: Scripts warn when time filtering results in no data points
- **Multi-Day Analysis**: Filters apply the same hour range to each day in the collection period
- **Project Filtering**: Process only specific projects by name or ID
- **Cluster Filtering**: Focus on individual clusters (requires project filter for organization-wide script)

### Usage Examples
- `--time-filter-start 09:00 --time-filter-end 17:00` - Standard business hours
- `--time-filter-start 14:00 --time-filter-end 23:59` - Afternoon/evening peak hours  
- `--time-filter-start 22:00 --time-filter-end 06:00` - Night shift (cross-midnight)
- `--time-filter-start 00:00 --time-filter-end 08:00` - Early morning hours

### Important Notes
- Both start and end times must be provided together for time filtering
- Time format must be `HH:MM` (24-hour format) 
- Without filters, all projects/clusters/timestamps are processed
- Project/cluster filters match by either ID or name (case-sensitive)
- Cluster filtering requires project filtering (for `atlas_metadata_collector.py`)
- Filtering occurs before metric aggregation (max/avg calculations)

## Getting MongoDB Atlas Credentials

1. Log in to MongoDB Atlas: https://cloud.mongodb.com/
2. Go to "Access Manager" → "API Keys"
3. Create an API key with read permissions
4. Copy the Public Key and Private Key
5. Get Organization ID from "Settings" → "Organization Settings"
6. Get Project ID from the project's URL or "Settings" → "Project Settings"

## License

This script is provided as-is for collecting MongoDB Atlas metadata. It is not supported by MongoDB, Inc. under any of their commercial support subscriptions or otherwise. Any usage of this script is at your own risk.

