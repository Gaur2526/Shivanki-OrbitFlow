# OrbitFlow Satellite Imaging Simulator

OrbitFlow is a small educational project for managing simulated satellite imaging tasks. It combines a FastAPI web dashboard, a MySQL database, a Python scheduling engine, and a C++ terminal application. The project demonstrates task scheduling, resource allocation, deadlock detection, and browsing images from satellite, fire, and traffic datasets.

> **Project status:** This repository contains application scripts, but does not include a database schema or seed data, a dependency lock file, or the optional `dataset/` files. The MySQL database and its core tables must be prepared separately before the dashboard and database-backed features can be used.

## Features

- Create and browse imaging tasks associated with regions and routes.
- Schedule tasks using First Come First Served (FCFS), non-preemptive Shortest Job First (SJF), non-preemptive Priority, and Round Robin (RR).
- Compare scheduling metrics: waiting time, turnaround time, response time, and completion time.
- Run a simulated mission that updates task state and progress in MySQL.
- View satellite, route, region, image, and activity-log information in a browser dashboard.
- Demonstrate resource allocation and deadlock detection. These are controlled simulations, not live satellite or operating-system resource telemetry.
- Use the C++ menu-driven application to add tasks and run scheduling/resource demonstrations.
- Import supported image files and generate previews when the optional datasets are available.

## Project layout

| File | Purpose |
| --- | --- |
| `dashboard.py` | FastAPI application, API endpoints, embedded dashboard UI, and mission simulation. |
| `scheduler.py` | Python FCFS, SJF, Priority, and Round Robin scheduling algorithms. |
| `deadlock_sim.py` | Banker's-style deadlock detection logic and a controlled demo scenario. |
| `db_config.py` | Local MySQL connection settings. Update this for your machine; do not publish real credentials. |
| `db_bridge.py` | MySQL bridge used by the C++ application to load and add tasks. |
| `main.cpp` | C++ terminal application. |
| `inspect_db.py` | Prints database tables, columns, row counts, and sample rows. |
| `import_dataset.py` | Imports supported GeoTIFF images/masks and generates PNG previews. |
| `import_fire_dataset.py` | Imports up to 50 CEMS fire-dataset PNG files. |
| `import_traffic_dataset.py` | Imports JPG/PNG traffic images. |
| `merge_cems.py` | Joins the 11 expected split CEMS archive parts into an archive. |
| `preview_satellite.py` | Creates satellite image and mask previews from configured sample filenames. |
| `preview_fire.py`, `preview_traffic.py` | Display sample fire or traffic images with Matplotlib. |
| `view_satellite.py` | Tkinter image viewer backed by MySQL image records. |

## Requirements

- Python 3.10 or newer is recommended.
- MySQL Server 8.x (or a compatible MySQL server), running locally or remotely.
- A C++17 compiler such as `g++` (only needed for the terminal application).
- Optional image data files, if using the dataset import and preview tools.

## 1. Get the project and create a Python environment

Run these commands from the project directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install fastapi uvicorn mysql-connector-python
```

The image importer and TIFF gallery preview need additional packages:

```bash
python -m pip install numpy rasterio pillow
```

The standalone preview scripts use Matplotlib:

```bash
python -m pip install matplotlib
```

On Windows, activate the environment with `.venv\Scripts\activate` and use `python` in place of `python3` where appropriate.

## 2. Configure and prepare MySQL

Edit `db_config.py` and set the host, username, password, database name, and port for your MySQL server. The checked-in file currently points to a local database named `orbitflow` and has example credentials. Replace those values with your own local settings.

Create the database if it does not already exist:

```sql
CREATE DATABASE orbitflow CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Select it in your MySQL client before creating/importing the application's tables:

```sql
USE orbitflow;
```

### Database schema prerequisite

The repository does **not** contain SQL to create the core tables or sample rows. The application expects these tables and columns to exist:

- `regions`: `region_id`, `region_name`
- `satellites`: `satellite_id` (other columns may be displayed by the dashboard)
- `routes`: `route_id`, `route_name`, `satellite_id`
- `imaging_tasks`: `task_id`, `task_name`, `region_id`, `satellite_id`, `route_id`, `arrival_time`, `burst_time`, `priority`, `status`, `resources_required`
- `dataset_images`: `image_id`, `image_name`, `image_type`, `image_path`; image importers also use `task_id`
- `scheduling_logs`: optional; the dashboard attempts a best-effort insert and adapts to its columns
- `resource_logs`: optional; the dashboard reads this table if present

Foreign keys between tasks, regions, routes, and satellites are appropriate but not required by the scripts. Add region, satellite, and route rows before creating tasks: task creation validates these references. The dashboard adds `imaging_tasks.progress`, `dataset_images.region_name`, `dataset_images.task_id`, and an `activity_log` table on startup when they are missing. It does not create the other core tables or populate reference data.

If you were given a database export or schema by the project author, import it into `orbitflow` before starting the dashboard. Otherwise, the database schema and initial region/satellite/route records need to be created for your own environment. Check connectivity and inspect an existing database with:

```bash
python inspect_db.py
```

## 3. Start the web dashboard

With MySQL running, the connection settings correct, and the required tables present:

```bash
source .venv/bin/activate
uvicorn dashboard:app --reload
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000). The interactive API documentation is at [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs).

To use a different port or make the server visible to other devices on your network:

```bash
uvicorn dashboard:app --reload --host 0.0.0.0 --port 8000
```

The mission simulator runs tasks sequentially and uses a simulated pool of five resource units. A task requesting more than five units is marked `Failed`. Mission progress is stored in the task's `progress` column. Stopping the server mid-run may leave tasks in an active state until the next dashboard startup, which resets `Scheduled` and `Processing` tasks to `Pending`.

## 4. Start the C++ terminal application (optional)

Build from the project directory:

```bash
g++ -std=c++17 -O2 -o orbitflow main.cpp
./orbitflow
```

The program loads and writes task records through `db_bridge.py`, so MySQL, `db_config.py`, and the core schema/reference rows must be ready first.

### macOS/Linux Python command note

The C++ source invokes `py db_bridge.py ...` and `python view_satellite.py`. The `py` launcher is common on Windows but may not be installed on macOS/Linux. If needed, edit those command strings in `main.cpp` to use `python3` (or the path to the Python interpreter in your active virtual environment), then rebuild. The C++ image-viewer option also requires Pillow and a graphical desktop session.

## Dataset tools

Dataset assets are not included in this repository. Place authorized data under the paths expected by the scripts. Keep the original data files unchanged and do not commit large or restricted datasets unless you have permission.

### GeoTIFF satellite and mask images

`import_dataset.py` scans the `dataset/` tree for `.tif` and `.tiff` files. It categorizes names containing `S1`, `S2`, or `Label`, creates display PNGs under `dataset/previews/`, and records them in `dataset_images`. Unknown filenames are skipped. Run a preview/dry run first:

```bash
python import_dataset.py --dry-run
python import_dataset.py
```

The script can scan a different dataset root with `--root /path/to/dataset`. Its classification expects recognizable file names and its saved paths are written relative to the project, so using a root outside the project may need path adjustments for the dashboard to serve those files.

`preview_satellite.py` expects specific example files under `dataset/images/` and `dataset/masks/`; change the filenames in that script to match your data if necessary.

### Fire data

`merge_cems.py` expects exactly 11 parts named `train.tar.0000.gz.part` through `train.tar.0010.gz.part` under `dataset/OrbitFlow_Collection/Fire/CEMS_Train` (possibly in nested directories). It creates `assembled/train.tar.gz`. Extract the archive to produce the `assembled/train` image folder if required by the other scripts.

Then preview or import images:

```bash
python preview_fire.py
python import_fire_dataset.py
```

The importer reads PNG files from `dataset/OrbitFlow_Collection/Fire/CEMS_Train/assembled/train` and imports up to 50.

### Traffic data

Traffic scripts expect images below `dataset/OrbitFlow_Collection/Traffic/DLR-MVDA`:

```bash
python preview_traffic.py
python import_traffic_dataset.py
```

The importers skip rows already present by image name or path. Importing requires a working `dataset_images` table.

## Scheduling algorithms

| Algorithm | Behavior |
| --- | --- |
| FCFS | Runs arrived tasks in arrival order. |
| SJF | Selects the shortest burst among arrived tasks; non-preemptive. |
| Priority | Selects the lowest numeric priority among arrived tasks (1 is highest); non-preemptive. |
| RR | Runs tasks in time slices using a configurable positive quantum. |

The dashboard's schedule API can calculate plans without starting a mission. The mission start action processes pending tasks and persists status/progress changes. Scheduling times are simulation units; dashboard mission playback uses a fixed delay per burst unit.

## Troubleshooting

- **MySQL connection error:** Check that MySQL is running, the database exists, and `db_config.py` credentials/host/port are correct.
- **Unknown table or column:** The core schema is not included. Import or create the required tables and seed the `regions`, `routes`, and `satellites` lookup data.
- **Dashboard starts but shows disconnected:** Check the server terminal for the database error and verify access from the machine running Uvicorn.
- **Image not found:** Confirm the file exists at the stored `image_path`. Paths are resolved relative to the project directory unless absolute.
- **No images found:** Dataset files are optional and are not distributed with this code; verify folder names and expected file extensions.
- **C++ cannot start Python bridge:** Replace `py` in `main.cpp` with the correct Python executable for your operating system and rebuild.
- **Task creation cannot find route or region:** Add those rows to the lookup tables first and use their IDs in the dashboard.

## Safety and limitations

- `db_config.py` contains local connection settings. Use private local credentials and do not share production secrets.
- Dashboard CORS is configured to allow all origins; restrict it before exposing the application beyond local development.
- Resource allocation, orbital visualization, priority suggestions, and deadlock examples are educational simulations. They do not represent live satellite telemetry, real resource allocation, or image recognition.
- The dashboard's startup schema changes are additive, but it also resets tasks left `Scheduled` or `Processing` back to `Pending`.

## License and data attribution

No license or dataset attribution file is included in this repository. Confirm usage rights and add the appropriate license and dataset credits before redistributing the code or data.
