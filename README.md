# gcstoazureblob

GCS → Pub/Sub → Azure Blob Sync
GCP Configuration Steps
1. Project Configuration
Set the active project:
gcloud config set project gcp-poc-501612
gcloud config get-value project
Output -> gcp-poc-501612
2. Ensure Pub/Sub API is Enabled
gcloud services enable pubsub.googleapis.com
Verify:
gcloud services list --enabled | grep pubsub
Expected Output:
pubsub.googleapis.com
3. Create Pub/Sub Topic
gcloud pubsub topics create gcs-azureblob
Verify:
gcloud pubsub topics list
Expected:
gcs-azureblob
4. Create Pub/Sub Subscription
gcloud pubsub subscriptions create gcs-azureblob-sync --topic=gcs-azureblob
Verify:
gcloud pubsub subscriptions list
Expected:
gcs-azureblob-sync
5. Configure GCS Bucket Notifications
Bucket:
highradius-poc-gcs-blobstorage
Create Notification for OBJECT_FINALIZE
gcloud storage buckets notifications create gs://highradius-poc-gcs-blobstorage 
--topic=gcs-azureblob --event-types=OBJECT_FINALIZE
Create Notification for OBJECT_DELETE
gcloud storage buckets notifications create gs://highradius-poc-gcs-blobstorage 
--topic=gcs-azureblob --event-types=OBJECT_DELETE
Create Notification for OBJECT_METADATA_UPDATE
gcloud storage buckets notifications create gs://highradius-poc-gcs-blobstorage 
--topic=gcs-azureblob --event-types=OBJECT_METADATA_UPDATE
Create Notification for OBJECT_ARCHIVE
gcloud storage buckets notifications create gs://highradius-poc-gcs-blobstorage --topic=gcs-azureblob --event-types=OBJECT_ARCHIVE
Event Coverage
OBJECT_FINALIZE
Triggered when:
New file upload
Existing file overwrite
OBJECT_DELETE
Triggered when:
File deleted
OBJECT_METADATA_UPDATE
Triggered when:
Content-Type change
Labels change
Custom metadata update
Storage class change
OBJECT_ARCHIVE
Triggered when:
Object archived
Versioning event
Previous object generation replaced
Verify Notifications
gcloud storage buckets notifications list gs://highradius-poc-gcs-blobstorage
Expected:
OBJECT_FINALIZE
OBJECT_DELETE
OBJECT_METADATA_UPDATE
OBJECT_ARCHIVE
6. Create Pub/Sub Listener Service Account
gcloud iam service-accounts create pubsub-listener-sa --display-name="PubSub Listener Service Account"
7. Grant Pub/Sub Subscriber Access (Enterprise Recommendation)
Subscription-Level Access
gcloud pubsub subscriptions add-iam-policy-binding gcs-azureblob-sync --member="serviceAccount:pubsub-listener-sa@gcp-poc-501612.iam.gserviceaccount.com" --role="roles/pubsub.subscriber"
Purpose
Read Pub/Sub messages
Pull bucket notifications
Acknowledge messages

8. Grant GCS Read Access
Bucket-Level Access
gcloud storage buckets add-iam-policy-binding gs://highradius-poc-gcs-blobstorage --member="serviceAccount:pubsub-listener-sa@gcp-poc-501612.iam.gserviceaccount.com" --role="roles/storage.objectViewer"
Purpose
Read files from GCS
Download objects
Access object metadata
9. Create Key for Listener Service Account
gcloud iam service-accounts keys create pubsub-listener-sa-key.json --iam-account=pubsub-listener-sa@gcp-poc-501612.iam.gserviceaccount.com
This key will be used by the Python listener service running on Azure VM.
10. Create Rclone Service Account
gcloud iam service-accounts create rclone-sa --display-name="Rclone Service Account"
11. Grant Rclone Access
Bucket-Level Access
gcloud storage buckets add-iam-policy-binding gs://highradius-poc-gcs-blobstorage --member="serviceAccount:rclone-sa@gcp-poc-501612.iam.gserviceaccount.com" --role="roles/storage.objectAdmin"
Purpose
Read objects
Create objects
Update objects
Delete objects
Manage GCS files
12. Create Key for Rclone Service Account
gcloud iam service-accounts keys create rclone-sa-key.json --iam-account=rclone-sa@gcp-poc-501612.iam.gserviceaccount.com
13. Verify Current Setup
Verify Topics
gcloud pubsub topics list
Verify Subscriptions
gcloud pubsub subscriptions list
Verify Notifications
gcloud storage buckets notifications list gs://highradius-poc-gcs-blobstorage

Verify Service Accounts
gcloud iam service-accounts list
Verify JSON Keys
ls -ltr *.json
Expected:
pubsub-listener-sa-key.json
rclone-sa-key.json

Final Enterprise Security Model
pubsub-listener-sa
Permissions:
Subscription:
gcs-azureblob-sync
└── roles/pubsub.subscriber

Bucket:
highradius-poc-gcs-blobstorage
└── roles/storage.objectViewer
Capabilities:
Read Pub/Sub messages
Read files from GCS
Cannot modify files
Cannot access other buckets
rclone-sa
Permissions:
Bucket:
highradius-poc-gcs-blobstorage
└── roles/storage.objectAdmin
Capabilities:
Read files
Create files
Update files
Delete files
Cannot access other buckets

Final Architecture
Google Cloud Storage
(highradius-poc-gcs-blobstorage)
│
▼
Pub/Sub Topic
(gcs-azureblob)
│
▼
Subscription
(gcs-azureblob-sync)
│
▼
pubsub-listener-sa
│
▼
Python Listener Service
│
▼
rclone-sa
│
▼
Azure Blob Storage

Azure VM Configuration
1. Create Working Directory
sudo mkdir -p /opt/gcs-azure-sync
sudo chown -R trianzadmin:trianzadmin /opt/gcs-azure-sync
cd /opt/gcs-azure-sync
2. Copy Service Account Keys
Copy from GCP Cloud Shell to VM:
pubsub-listener-sa-key.json
rclone-sa-key.json
Directory:
/opt/gcs-azure-sync/
├── pubsub-listener-sa-key.json
├── rclone-sa-key.json
Set permissions:
chmod 600 *.json
Step 1: Install Python 
sudo apt update
sudo apt install -y python3-full python3-venv
Then verify:
python3 -m venv venv
Now activate it:
source venv/bin/activate
Expected prompt:
(venv) trianzadmin@gcstoblobstorage:/opt/gcs-azure-sync$
Install Python Packages
pip install --upgrade pip
pip install google-cloud-pubsub google-cloud-storage azure-storage-blob python-dotenv
Verify:
pip list
Test Pub/Sub Authentication
export GOOGLE_APPLICATION_CREDENTIALS=/opt/gcs-azure-sync/pubsub-listener-sa-key.json
Install and configure Rclone:
curl https://rclone.org/install.sh | sudo bash
Verify Rclone Configuration
Check Configured Remotes
rclone listremotes
Expected Output:
gcs:
azureblob:
Display Rclone Configuration
rclone config show
Expected Output:
[gcs]
type = google cloud storage
project_number = 615629292552
service_account_file = /opt/gcs-azure-sync/rclone-sa-key.json
bucket_policy_only = true
location = asia-south1

[azureblob]
type = azureblob
account = hirteststoragemigration
key = ***************

RCLONE CONFIGURATION RUNBOOK

Objective
---------

Configure Rclone on Azure VM to transfer files between Google Cloud Storage (GCS) and Azure Blob Storage.

Prerequisites
-------------

Verify service account keys are available:

ls -ltr /opt/gcs-azure-sync/

Expected:

pubsub-listener-sa-key.json
rclone-sa-key.json

Verify Rclone Installation
--------------------------

rclone version

Expected:

rclone v1.x.x

Configure GCS Remote
--------------------

Start configuration:

rclone config

Select:

n) New remote

Remote Name:

gcs

Storage Type:

Google Cloud Storage

Service Account File:

/opt/gcs-azure-sync/rclone-sa-key.json

Project Number:

615629292552

Bucket Policy Only:

true

Location:

asia-south1

Advanced Configuration:

n

Confirm Save:

y

Expected Configuration:

[gcs]
type = google cloud storage
project_number = 615629292552
service_account_file = /opt/gcs-azure-sync/rclone-sa-key.json
bucket_policy_only = true
location = asia-south1

Configure Azure Blob Remote
---------------------------

Run:

rclone config

Select:

n) New remote

Remote Name:

azureblob

Storage Type:

Microsoft Azure Blob Storage

Storage Account Name:

hirteststoragemigration

Storage Account Key:

<Azure Storage Account Key>

Advanced Configuration:

n

Confirm Save:

y

Expected Configuration:

[azureblob]
type = azureblob
account = hirteststoragemigration
key = **************

Verify Configured Remotes
-------------------------

rclone listremotes

Expected:

gcs:
azureblob:

Verify Configuration File
-------------------------

rclone config file

Expected:

Configuration file is stored at:

/home/trianzadmin/.config/rclone/rclone.conf

Display Configuration
---------------------

rclone config show

Expected:

[gcs]
type = google cloud storage
project_number = 615629292552
service_account_file = /opt/gcs-azure-sync/rclone-sa-key.json
bucket_policy_only = true
location = asia-south1

[azureblob]
type = azureblob
account = hirteststoragemigration
key = **************

Validate GCS Connectivity
-------------------------

List bucket contents:

rclone lsd gcs:highradius-poc-gcs-blobstorage

List files:

rclone ls gcs:highradius-poc-gcs-blobstorage

Validate Azure Blob Connectivity
--------------------------------

List containers:

rclone lsd azureblob:

List files in container:

rclone ls azureblob:<container-name>

Example:

rclone ls azureblob:gcsdemo

Test Copy from GCS to Azure Blob
--------------------------------

Copy a single file:

rclone copy \
gcs:highradius-poc-gcs-blobstorage/harigcs-azureblob.txt \
azureblob:<container-name> \
-v

Verify File in Azure Blob
-------------------------

rclone ls azureblob:<container-name>

Dry Run Test
------------

rclone copy \
gcs:highradius-poc-gcs-blobstorage \
azureblob:<container-name> \
--dry-run \
-vv

Full Synchronization Test
-------------------------

rclone sync \
gcs:highradius-poc-gcs-blobstorage \
azureblob:<container-name> \
-v

Verify:

rclone ls azureblob:<container-name>

Configuration Summary
---------------------

Source

Remote Name : gcs
Bucket : highradius-poc-gcs-blobstorage
Project : gcp-poc-501612
Authentication : Service Account

Pub/Sub Listener Service Runbook
Objective
Create a Python service on Azure VM that:








# gcstoazureblob

GCS → Pub/Sub → Azure Blob Sync (Production Runbook)
1. Solution Overview
Source
Plain Text
1
Google Cloud Storage (GCS)
2
Bucket:
3
highradius-poc-gcs-blobstorage
Show more lines
Event Layer
Plain Text
1
Pub/Sub Topic:
2
gcs-azureblob
3
 
4
Pub/Sub Subscription:
5
gcs-azureblob-sync
Show more lines
Processing Layer
Plain Text
1
Azure VM:
2
gcstoblobstorage
3
 
4
OS:
5
Ubuntu 24.04
6
 
7
VM Size:
8
Standard_B2als_v2
Show more lines
Metadata Store
Plain Text
1
PostgreSQL
2
 
3
Database:
4
gcssync
5
 
6
User:
7
gcssync
Show more lines
Target
Plain Text
1
Azure Blob Storage
2
 
3
Storage Account:
4
hirteststoragemigration
5
 
6
Container:
7
testcontainer
Show more lines
2. Architecture
Plain Text
1
GCS Bucket
2
│
3
▼
4
Pub/Sub Topic
5
│
6
▼
7
Pub/Sub Subscription
8
│
9
▼
10
Azure VM Listener
11
│
12
▼
13
PostgreSQL
14
│
15
▼
16
Rclone
17
│
18
▼
19
Azure Blob Storage
Show more lines
3. Install Required Packages
Shell
1
sudo apt update
2
 
3
sudo apt install -y \
4
python3 \
5
python3-pip \
6
python3-venv \
7
postgresql \
8
postgresql-contrib \
9
sqlite3
Show more lines
4. Create Application Directory
Shell
1
sudo mkdir -p /opt/gcs-azure-sync
2
 
3
sudo chown trianzadmin:trianzadmin \
4
/opt/gcs-azure-sync
Show more lines
5. Create Python Virtual Environment
Shell
1
cd /opt/gcs-azure-sync
2
 
3
python3 -m venv venv
Show more lines

Activate:

Shell
1
source venv/bin/activate
Show more lines
6. Install Python Packages
Shell
1
pip install --upgrade pip
2
 
3
pip install google-cloud-pubsub
4
 
5
pip install psycopg2-binary
6
 
7
pip install google-cloud-storage
Show more lines
7. Rclone Configuration

Verify:

Shell
1
rclone config show
Show more lines

Expected:

INI
1
[gcs]
2
type = google cloud storage
3
project_number = 615629292552
4
service_account_file = /opt/gcs-azure-sync/rclone-sa-key.json
5
 
6
[azureblob]
7
type = azureblob
8
account = hirteststoragemigration
9
key = <storage-account-key>
Show more lines
8. PostgreSQL Setup

Login:

Shell
1
sudo -u postgres psql
Show more lines

Create Database:

SQL
1
CREATE DATABASE gcssync;
Show more lines

Create User:

SQL
1
CREATE USER gcssync
2
WITH PASSWORD 'Passw0rd@123';
Show more lines

Grant:

SQL
1
GRANT ALL PRIVILEGES
2
ON DATABASE gcssync
3
TO gcssync;
Show more lines

Exit:

SQL
1
\q
Show more lines
9. Create PostgreSQL Table
Shell
1
sudo -u postgres psql -d gcssync
Show more lines
SQL
1
CREATE TABLE IF NOT EXISTS file_events
2
(
3
event_id VARCHAR(100) PRIMARY KEY,
4
bucket_name VARCHAR(255),
5
object_name TEXT,
6
generation VARCHAR(100),
7
status VARCHAR(30),
8
retry_count INT DEFAULT 0,
9
error_message TEXT,
10
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
11
completed_at TIMESTAMP
12
);
Show more lines

Indexes:

SQL
1
CREATE INDEX idx_status
2
ON file_events(status);
3
 
4
CREATE INDEX idx_created_at
5
ON file_events(created_at);
6
 
7
CREATE INDEX idx_fileevents_object_name
8
ON file_events(object_name);
Show more lines

Grant Permissions:

SQL
1
GRANT ALL PRIVILEGES
2
ON TABLE file_events
3
TO gcssync;
4
 
5
GRANT USAGE
6
ON SCHEMA public
7
TO gcssync;
8
 
9
GRANT ALL PRIVILEGES
10
ON ALL SEQUENCES IN SCHEMA public
11
TO gcssync;
12
 
13
ALTER TABLE file_events
14
OWNER TO gcssync;
Show more lines

Exit:

SQL
1
\q
Show more lines
10. Environment File

Create:

Shell
1
nano /opt/gcs-azure-sync/.env
Show more lines

Contents:

INI
1
GCP_PROJECT_ID=gcp-poc-501612
2
 
3
PUBSUB_SUBSCRIPTION_ID=gcs-azureblob-sync
4
 
5
RCLONE_CONFIG=/home/trianzadmin/.config/rclone/rclone.conf
6
 
7
RCLONE_SOURCE_REMOTE=gcs
8
 
9
SOURCE_BUCKET=highradius-poc-gcs-blobstorage
10
 
11
RCLONE_TARGET_REMOTE=azureblob
12
 
13
TARGET_CONTAINER=testcontainer
14
 
15
PGHOST=localhost
16
PGPORT=5432
17
 
18
PGDATABASE=gcssync
19
 
20
PGUSER=gcssync
21
 
22
PGPASSWORD=Passw0rd@123
23
 
24
WORKERS=8
25
 
26
RCLONE_BIN=/usr/bin/rclone
27
 
28
RCLONE_TIMEOUT=3600s
29
 
30
GOOGLE_APPLICATION_CREDENTIALS=/opt/gcs-azure-sync/pubsub-listener-sa-key.json
Show more lines

Secure:

Shell
1
chmod 600 /opt/gcs-azure-sync/.env
Show more lines
11. Production Listener Script

Create:

Shell
1
nano /opt/gcs-azure-sync/listener.py
Show more lines

Paste:

Python
1
#!/usr/bin/env python3
2
 
3
import json
4
import logging
5
import os
6
import signal
7
import psycopg2
8
import subprocess
9
import threading
10
 
11
from google.cloud import pubsub_v1
12
 
13
PROJECT_ID = os.environ["GCP_PROJECT_ID"]
14
SUBSCRIPTION_ID = os.environ["PUBSUB_SUBSCRIPTION_ID"]
15
 
16
SOURCE_REMOTE = os.environ.get("RCLONE_SOURCE_REMOTE", "gcs")
17
SOURCE_BUCKET = os.environ["SOURCE_BUCKET"]
18
 
19
TARGET_REMOTE = os.environ.get("RCLONE_TARGET_REMOTE", "azureblob")
20
TARGET_CONTAINER = os.environ["TARGET_CONTAINER"]
21
 
22
PGHOST = os.environ["PGHOST"]
23
PGPORT = os.environ["PGPORT"]
24
PGDATABASE = os.environ["PGDATABASE"]
25
 
26
PGUSER = os.environ["PGUSER"]
27
PGPASSWORD = os.environ["PGPASSWORD"]
28
 
29
WORKERS = int(os.environ.get("WORKERS", "8"))
30
 
31
RCLONE_BIN = os.environ.get(
32
"RCLONE_BIN",
33
"/usr/bin/rclone"
34
)
35
 
36
RCLONE_CONFIG = os.environ.get(
37
"RCLONE_CONFIG",
38
"/home/trianzadmin/.config/rclone/rclone.conf"
39
)
40
 
41
RCLONE_TIMEOUT = os.environ.get(
42
"RCLONE_TIMEOUT",
43
"3600s"
44
)
45
 
46
logging.basicConfig(
47
level=logging.INFO,
48
format="%(asctime)s %(levelname)s %(threadName)s %(message)s",
49
)
50
 
51
log = logging.getLogger("gcs-rclone-sync")
52
 
53
stop_event = threading.Event()
54
db_lock = threading.Lock()
55
 
56
def db():
57
return psycopg2.connect(
58
host=PGHOST,
59
port=PGPORT,
60
dbname=PGDATABASE,
61
user=PGUSER,
62
password=PGPASSWORD
63
)
64
 
65
def already_done(event_id):
66
 
67
with db_lock:
68
 
69
conn = db()
70
cur = conn.cursor()
71
 
72
cur.execute(
73
"""
74
SELECT status
75
FROM file_events
76
WHERE event_id=%s
77
""",
78
(event_id,)
79
)
80
 
81
row = cur.fetchone()
82
 
83
cur.close()
84
conn.close()
85
 
86
return row is not None and row[0] == "COMPLETED"
87
 
88
def mark(event_id,bucket,obj,generation,status,error=None):
89
 
90
with db_lock:
91
 
92
conn = db()
93
cur = conn.cursor()
94
 
95
cur.execute(
96
"""
97
INSERT INTO file_events
98
(
99
event_id,
100
bucket_name,
101
object_name,
102
generation,
103
status,
104
error_message
105
)
106
VALUES
107
(
108
%s,%s,%s,%s,%s,%s
109
)
110
ON CONFLICT (event_id)
111
DO UPDATE SET
112
status=EXCLUDED.status,
113
error_message=EXCLUDED.error_message
114
""",
115
(
116
event_id,
117
bucket,
118
obj,
119
generation,
120
status,
121
error
122
)
123
)
124
 
125
conn.commit()
126
 
127
cur.close()
128
conn.close()
129
 
130
def transfer(bucket,obj):
131
 
132
src=f"{SOURCE_REMOTE}:{bucket}/{obj}"
133
dst=f"{TARGET_REMOTE}:{TARGET_CONTAINER}/{obj}"
134
 
135
cmd=[
136
RCLONE_BIN,
137
"copyto",
138
src,
139
dst,
140
"--config",
141
RCLONE_CONFIG,
142
"--retries","3",
143
"--low-level-retries","10",
144
"--timeout",RCLONE_TIMEOUT,
145
"--stats","30s",
146
"--log-level","INFO"
147
]
148
 
149
log.info("SOURCE=%s TARGET=%s",src,dst)
150
 
151
result=subprocess.run(
152
cmd,
153
capture_output=True,
154
text=True,
155
timeout=3720
156
)
157
 
158
if result.returncode!=0:
159
raise RuntimeError(
160
f"rclone failed rc={result.returncode}: "
161
f"{result.stderr[-4000:]}"
162
)
163
 
164
def process_message(message):
165
 
166
event_id=message.message_id
167
 
168
payload=json.loads(
169
message.data.decode("utf-8")
170
)
171
 
172
bucket=payload.get("bucket")
173
obj=payload.get("name")
174
 
175
generation=str(
176
payload.get("generation","")
177
)
178
 
179
if bucket!=SOURCE_BUCKET:
180
return True
181
 
182
if already_done(event_id):
183
return True
184
 
185
mark(
186
event_id,
187
bucket,
188
obj,
189
generation,
190
"PROCESSING"
191
)
192
 
193
try:
194
 
195
transfer(bucket,obj)
196
 
197
mark(
198
event_id,
199
bucket,
200
obj,
201
generation,
202
"COMPLETED"
203
)
204
 
205
log.info(
206
"Completed event=%s object=%s",
207
event_id,
208
obj
209
)
210
 
211
return True
212
 
213
except Exception as exc:
214
 
215
mark(
216
event_id,
217
bucket,
218
obj,
219
generation,
220
"FAILED",
221
str(exc)
222
)
223
 
224
log.exception(
225
"Transfer failed event=%s object=%s",
226
event_id,
227
obj
228
)
229
 
230
return False
231
 
232
def callback(message):
233
 
234
ack=process_message(message)
235
 
236
if ack:
237
message.ack()
238
else:
239
message.nack()
240
 
241
def shutdown(signum,frame):
242
stop_event.set()
243
 
244
signal.signal(signal.SIGTERM,shutdown)
245
signal.signal(signal.SIGINT,shutdown)
246
 
247
def main():
248
 
249
log.info(
250
"Starting listener project=%s subscription=%s workers=%s",
251
PROJECT_ID,
252
SUBSCRIPTION_ID,
253
WORKERS
254
)
255
 
256
subscriber=pubsub_v1.SubscriberClient()
257
 
258
subscription_path=(
259
subscriber.subscription_path(
260
PROJECT_ID,
261
SUBSCRIPTION_ID
262
)
263
)
264
 
265
flow_control=pubsub_v1.types.FlowControl(
266
max_messages=WORKERS,
267
max_bytes=WORKERS * 64 * 1024 * 1024
268
)
269
 
270
streaming_future=subscriber.subscribe(
271
subscription_path,
272
callback=callback,
273
flow_control=flow_control
274
)
275
 
276
streaming_future.result()
277
 
278
if __name__=="__main__":
279
main()
Show less
12. Syntax Validation
Shell
1
python -m py_compile \
2
/opt/gcs-azure-sync/listener.py
Show more lines

No output means success.

13. Manual Validation
Shell
1
source venv/bin/activate
2
 
3
set -a
4
source .env
5
set +a
6
 
7
python listener.py
Show more lines

Expected:

Plain Text
1
Starting listener project=gcp-poc-501612
2
subscription=gcs-azureblob-sync
3
workers=8
Show more lines
14. Systemd Service
Shell
1
sudo nano /etc/systemd/system/gcs-rclone-sync.service
Show more lines
INI
1
[Unit]
2
Description=GCS PubSub Rclone Sync
3
After=network.target
4
 
5
[Service]
6
Type=simple
7
User=trianzadmin
8
 
9
WorkingDirectory=/opt/gcs-azure-sync
10
 
11
EnvironmentFile=/opt/gcs-azure-sync/.env
12
 
13
ExecStart=/opt/gcs-azure-sync/venv/bin/python \
14
/opt/gcs-azure-sync/listener.py
15
 
16
Restart=always
17
 
18
RestartSec=10
19
 
20
[Install]
21
WantedBy=multi-user.target
Show more lines
15. Enable Service
Shell
1
sudo systemctl daemon-reload
2
 
3
sudo systemctl enable gcs-rclone-sync
4
 
5
sudo systemctl restart gcs-rclone-sync
Show more lines

Verify:

Shell
1
sudo systemctl status gcs-rclone-sync
Show more lines
16. Monitoring

Live Logs:

Shell
1
journalctl -u gcs-rclone-sync -f
Show more lines
17. PostgreSQL Monitoring

Completed:

SQL
1
SELECT count(*)
2
FROM file_events
3
WHERE status='COMPLETED';
Show more lines

Failed:

SQL
1
SELECT count(*)
2
FROM file_events
3
WHERE status='FAILED';
Show more lines

Recent:

SQL
1
SELECT object_name,
2
status,
3
created_at
4
FROM file_events
5
ORDER BY created_at DESC
6
LIMIT 20;
Show more lines
18. Final Production Outcome

✅ GCS Event Driven

✅ Pub/Sub Based

✅ Multi-Threaded (WORKERS=8)

✅ PostgreSQL Audit Tracking

✅ Duplicate Protection

✅ Failure Tracking

✅ Rclone Integration

✅ Azure Blob Target

✅ Systemd Managed

✅ Production Ready

✅ Future Scale-Out Ready (Multiple Listener VMs + Shared PostgreSQL)

Provide your feedback on BizChat
You said:
