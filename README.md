StatAgent-Kernel.ai-
🇦🇹 StatAgent – Sozialversicherung Statistics & QA Engine

StatAgent is an automated, high-performance statistics calculation and quality assurance platform designed for the Dachverband der österreichischen Sozialversicherungsträger. It processes raw employment delivery files (raw_deliveries) and pension reports (raw_pensions), applies deterministic counting rules, performs automated QA anomaly detection, and exposes interactive analytics via a local executive dashboard and CLI.

📄 Key Features & Compliance Highlights

Strict Deterministic Processing: Zero language-model dependency in statistic calculations. Two runs with the same data and rules yield identical numbers (COUNTING_CONDITIONS mapped via SQL over clean_period).

Verifiable Audit Trail: Human-in-the-loop approvals backed by database logs (audit_log) and Git commits tracking authorized names, dates, and target checksums.

Change Request (CR 1–3) Support:

CR 1: Harmonization of regular employment contracts and free service contracts (Arbeiter, Angestellte, Beamte).

CR 2: Transition from snapshot/reference-date counting to full-month coverage (any relationship active 
≥
1
 day in the reporting period).

CR 3: Ministry Special Delivery (Sonderauswertung) filtering individuals with 
≥
200
 days in ÖNACE Section C over a 12-month window.

Zero Fake Data Policy: All dashboard components, classifications (ÖNACE, Region), and quality reports stream live from PostgreSQL.

Safe Read-Only Query Explorer: Ad-hoc SQL execution guarded against destructive operations (DROP, DELETE, UPDATE).

🛠 Quick Start & Installation

Prerequisites

Python 3.10+

Docker Desktop (or a local PostgreSQL 16 instance)

Clone the Repository & Install Dependencies
cd statagent pip install -r requirements.txt

(Required packages: nicegui, pandas, psycopg2-binary, sqlalchemy, openpyxl)

File Structure Verification
Ensure raw delivery files are placed directly inside lieferungen/ as flat files or .csv.gz archives (do not keep them as unzipped sub-folders):

lieferungen/ ├── lieferung_2024-08.csv.gz ├── lieferung_2024-09.csv.gz └── ...

Database Initialization
Start the PostgreSQL database container:

docker compose down -v docker compose up -d

Verify that the container is ready:

docker logs -f statagent-db

Initialize system schemas, classifications, and default rules:

python run.py --init

🖥 Executive Dashboard (NiceGUI Web UI)

Start the interactive web portal:

python app_nicegui.py

Navigate to http://localhost:8080 in your browser to access:

📜 License & Compliance

Designed strictly in accordance with the technical guidelines and delivery criteria for the Austrian Social Insurance (Dachverband der Österreichischen Sozialversicherungen).
