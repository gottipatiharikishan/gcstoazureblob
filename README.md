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




Blob-GCS











































Work
Harikishan
M365 Copilot (Basic)



Auto


You said:
ok
Copilot said:
Great ✅

Since the Event Grid subscription is ready, the next step in the rollback architecture is to build the consumer side.

Next Phase Overview
Step 1: Create a Dedicated VM
I recommend not using your current VM (gcstoblobstorage) for rollback processing.

Recommended VM:

Reason:

This keeps both directions isolated.

Step 2: Build Working Directory
SSH into the VM:

Step 3: Python Runtime
Verify:

Step 4: Create Virtual Environment
Step 5: Install Packages
Step 6: Prepare PostgreSQL
Reuse existing PostgreSQL if desired, or new PostgreSQL instance.

Create table:

Step 7: Create GCP Service Account (already discussed)
Grant:

Generate key and copy it to:

Step 8: Create Environment File
Step 9: Build Queue Listener
The listener should:

BlobCreated
BlobDeleted
DirectoryCreated
DirectoryDeleted
BlobRenamed
DirectoryRenamed
Step 10: Create systemd Service
Enable:

Validation
Upload:

to:

Verify:

Expected:

The next major deliverable is the full listener.py for Azure Queue → PostgreSQL → GCS, including handling all 6 Event Grid event types.



You said:

Copilot said:
On this screen, for the architecture we've selected:

you should choose:

From the dropdown, select:

Then Azure will ask for:

Select:

Then:

Select:

Final Production Configuration
Event Subscription Name
Event Schema
Event Types
Client requirement:

Filters
Recommended:

Dead Lettering
Dead Letter Container:

Retry Policy
Expiration
Delivery Properties
Endpoint Type
Queue Destination
After Clicking Create
The flow becomes:

Once the Event Subscription is created successfully, the next step is:

and I'll give you:

PostgreSQL blob_events table
.env
Full listener.py
Complete systemd service
Validation procedure
for the rollback architecture.



You said:

Copilot said:
This screen looks correct. ✅

For your enterprise rollback architecture, I would use the following settings:

Endpoint Details
Cross-Tenant Delivery
Keep it disabled unless the storage account and queue are in different Azure tenants.

Managed Identity for Delivery
Current:

For your design:

Keep it as:

Because Event Grid can write directly to the Storage Queue in the same subscription without requiring a managed identity.

Event Types Selected
You mentioned the client wants all six:

That's acceptable, provided the listener handles each event type appropriately.

Before Clicking Create
Verify:

What Happens Next
After you click Create:

The next implementation step will be:

Build Queue Consumer
We'll create:

and then:

that:

Polls Azure Storage Queue
Reads Event Grid messages
Handles all 6 event types
Writes audit records to PostgreSQL (blob_events)
Uploads files to GCS
Deletes files from GCS when needed
Runs as a systemd service
That listener implementation is the next major deliverable after this Event Grid subscription is created successfully.



You said:

still create option not highlight 
Copilot said:
The Create button is still disabled because one of the required fields on another tab is not valid or not completed.

From your screenshot, the most likely causes are:

1. System Topic Name (Most Common)
Scroll back to the Basics tab and check:

Make sure it contains a value, for example:

or

This field is mandatory.

2. Filters Tab
Since you selected all 6 event types, go to:

Verify there are no validation errors.

Recommended:

If a filter value is partially entered or invalid, Azure disables Create.

3. Additional Features Tab
You enabled:

Check the Event Time To Live fields.

The screenshot earlier showed:

This can cause validation issues.

Set:

Or:

depending on what Azure accepts in your portal version.

4. Expiration Time
Make sure:

is unchecked.

Production subscriptions should not expire automatically.

5. Endpoint Validation
Confirm:

If the queue was recently created, click:

and reselect the queue.

Quick Check
Go back through the tabs in this order:

Look for any red asterisks (*) or red validation messages.

The two fields most likely blocking Create are:

or

Tell me:

and whether Enable expiration time is currently checked or unchecked, and I'll pinpoint the exact issue.



You said:

Copilot said:
The Basics tab looks valid.

The reason Create is still disabled is almost certainly on one of the other tabs:

Filters
Additional Features
Delivery Properties
Based on your earlier screenshot, the most likely issue is still in Additional Features.

Check 1: Event Subscription Expiration Time
Go to Additional Features and make sure:

is unchecked.

For production:

If it remains enabled, Azure may require a valid future timestamp and can block creation.

Check 2: Event Time To Live
If Retry Policies are enabled, verify the TTL fields are valid.

Recommended:

Avoid:

Some portal versions accept it, some trigger validation issues.

Check 3: Filters Tab
Open Filters.

Verify:

contains a valid path, for example:

or leave it completely blank.

Do not leave a partially entered value.

Check 4: Storage Queue Permissions
Sometimes Event Grid cannot validate the queue endpoint if queue permissions aren't resolved yet.

Click:

next to:

and reselect:

Quickest Way to Find the Error
Go through the tabs:

and look for:

Red text
Red borders
Red asterisks with warnings
Azure usually marks the exact field preventing creation.

Send Screenshots
Please send screenshots of:

Filters
Additional Features
Delivery Properties
and I'll pinpoint exactly what's blocking the Create button.



You said:
Dead-Lettering
Save events that cannot be delivered to storage. Learn more 
Enable dead-lettering
Cross-tenant delivery
Disabled
Storage Subscription
HiRAzurePOC2026
Storage Blob Container
eventgrid-deadletter
 (change) 
Managed Identity for Dead-Letter
Managed identity type
None
Retry Policies
Customize how many times and for how long event delivery will be retried. Learn more 
Configure Retry Policies
Events will be dead-lettered or discarded after either the number of delivery attempts is exceeded or the time to live has elapsed.
Max Event Delivery Attempts
Event Time to Live
Days
Hours
Minutes
Seconds
Event Subscription Expiration Time
Set a time at which the event subscription will automatically be deleted.
Enable expiration time
Labels
Set metadata on this event subscription that is only visible through this experience. The metadata (labels) set does not affect the data plane behavior in any way. For example, labels do not show anywhere in the request or in the events that are sent to subscribers.



Copilot said:
I found the issue. ✅

The Create button is disabled because under Retry Policies, you've enabled:

but you haven't filled the required values:

Azure treats these as mandatory when retry policies are enabled.

Recommended Values
Max Event Delivery Attempts
Event Time To Live
Set:

This gives:

which is a good production value.

Event Subscription Expiration Time
Make sure:

is:

Do not enable expiration in production.

Dead Lettering
Current values are good:

Labels (Optional)
You can add:

but these are optional and do not affect functionality.

Final Additional Features Configuration
After entering the retry values, the Create button should become active.

Once the Event Grid Subscription is created successfully, we'll move to:

and continue building the rollback path:






Message Copilot


AI-generated content may be incorrect









































Work
Harikishan
M365 Copilot (Basic)



Auto


You said:
ok
Copilot said:
Great ✅

Since the Event Grid subscription is ready, the next step in the rollback architecture is to build the consumer side.

Next Phase Overview



Plain Text
Azure Blob Storage
        ↓
Event Grid
        ↓
Storage Queue (blob-sync-events)
        ↓
Azure VM Listener
        ↓
PostgreSQL (blob_events)
        ↓
Google Cloud Storage
Step 1: Create a Dedicated VM
I recommend not using your current VM (gcstoblobstorage) for rollback processing.

Recommended VM:




Plain Text
VM Name:
blobtogcslistener
 
OS:
Ubuntu 24.04 LTS
 
Region:
South India
 
Size:
Standard_D4s_v5
Reason:




Plain Text
Current VM:
GCS → Azure
 
New VM:
Azure → GCS (Rollback)
This keeps both directions isolated.

Step 2: Build Working Directory
SSH into the VM:




Shell
sudo mkdir -p /opt/blob-to-gcs
 
sudo chown -R $USER:$USER /opt/blob-to-gcs
 
cd /opt/blob-to-gcs
Step 3: Python Runtime



Shell
sudo apt update
 
sudo apt install -y \
python3 \
python3-pip \
python3-venv
Verify:




Shell
python3 --version
Step 4: Create Virtual Environment



Shell
python3 -m venv venv
 
source venv/bin/activate
Step 5: Install Packages



Shell
pip install --upgrade pip



Shell
pip install azure-storage-queue



Shell
pip install azure-storage-blob



Shell
pip install google-cloud-storage



Shell
pip install psycopg2-binary



Shell
pip install python-dotenv
Step 6: Prepare PostgreSQL
Reuse existing PostgreSQL if desired, or new PostgreSQL instance.

Create table:




SQL
CREATE TABLE blob_events
(
    event_id VARCHAR(100) PRIMARY KEY,
    blob_name TEXT,
    event_type VARCHAR(50),
    status VARCHAR(20),
    retry_count INTEGER DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP
);
Step 7: Create GCP Service Account (already discussed)



Shell
gcloud iam service-accounts create azure-to-gcs-sa \
    --display-name="Azure To GCS Service Account"
Grant:




Shell
gcloud storage buckets add-iam-policy-binding \
gs://highradius-poc-gcs-blobstorage \
--member="serviceAccount:azure-to-gcs-sa@gcp-poc-501612.iam.gserviceaccount.com" \
--role="roles/storage.objectAdmin"
Generate key and copy it to:




Plain Text
/opt/blob-to-gcs/azure-to-gcs-sa-key.json
Step 8: Create Environment File



Shell
nano /opt/blob-to-gcs/.env



INI
AZURE_STORAGE_ACCOUNT=hirteststoragemigration
 
AZURE_STORAGE_KEY=<REDACTED>
 
AZURE_QUEUE_NAME=blob-sync-events
 
GCP_BUCKET=highradius-poc-gcs-blobstorage
 
GOOGLE_APPLICATION_CREDENTIALS=/opt/blob-to-gcs/azure-to-gcs-sa-key.json
 
PGHOST=localhost
PGPORT=5432
 
PGDATABASE=gcssync
 
PGUSER=gcssync
 
PGPASSWORD=<REDACTED>
 
WORKERS=8

Show more lines
Step 9: Build Queue Listener
The listener should:

BlobCreated



Plain Text
Read Queue Event
       ↓
Download Blob
       ↓
Upload To GCS
       ↓
Update PostgreSQL
BlobDeleted



Plain Text
Read Queue Event
       ↓
Delete Object In GCS
       ↓
Update PostgreSQL
DirectoryCreated



Plain Text
Log Event
DirectoryDeleted



Plain Text
Log Event
BlobRenamed



Plain Text
Copy New Name
Delete Old Name
DirectoryRenamed



Plain Text
Record Rename Event
Step 10: Create systemd Service



Shell
sudo nano /etc/systemd/system/blob-to-gcs.service



INI
[Unit]
Description=Azure Blob To GCS Sync
After=network.target
 
[Service]
Type=simple
 
User=trianzadmin
 
WorkingDirectory=/opt/blob-to-gcs
 
EnvironmentFile=/opt/blob-to-gcs/.env
 
ExecStart=/opt/blob-to-gcs/venv/bin/python \
/opt/blob-to-gcs/listener.py
 
Restart=always
 
RestartSec=10
 
[Install]
WantedBy=multi-user.target

Show more lines
Enable:




Shell
sudo systemctl daemon-reload
 
sudo systemctl enable blob-to-gcs
 
sudo systemctl start blob-to-gcs
Validation
Upload:




Plain Text
sample.txt
to:




Plain Text
testcontainer
Verify:




Shell
gcloud storage ls \
gs://highradius-poc-gcs-blobstorage
Expected:




Plain Text
sample.txt
The next major deliverable is the full listener.py for Azure Queue → PostgreSQL → GCS, including handling all 6 Event Grid event types.





You said:

Copilot said:
On this screen, for the architecture we've selected:




Plain Text
Azure Blob
    ↓
Event Grid
    ↓
Storage Queue (blob-sync-events)
    ↓
Listener VM
    ↓
PostgreSQL
    ↓
Google Cloud Storage
you should choose:




Plain Text
Endpoint Type = Storage Queue
From the dropdown, select:




Plain Text
Storage Queue
Then Azure will ask for:




Plain Text
Storage Account
Select:




Plain Text
hirteststoragemigration
Then:




Plain Text
Queue
Select:




Plain Text
blob-sync-events
Final Production Configuration
Event Subscription Name



Plain Text
blob-to-gcs-eventgrid
Event Schema



Plain Text
Event Grid Schema
Event Types
Client requirement:




Plain Text
✅ Blob Created
✅ Blob Deleted
✅ Directory Created
✅ Directory Deleted
✅ Blob Renamed
✅ Directory Renamed
Filters
Recommended:




Plain Text
Subject Begins With
 
/blobServices/default/containers/testcontainer/
Dead Lettering



Plain Text
✅ Enabled
Dead Letter Container:




Plain Text
eventgrid-deadletter
Retry Policy



Plain Text
✅ Enabled
 
Max Delivery Attempts: 30
 
TTL: 1440 Minutes
Expiration



Plain Text
❌ Disabled
Delivery Properties



Plain Text
Leave Empty
Endpoint Type



Plain Text
✅ Storage Queue
Queue Destination



Plain Text
Storage Account:
hirteststoragemigration
 
Queue:
blob-sync-events
After Clicking Create
The flow becomes:




Plain Text
Azure Blob (testcontainer)
          │
          ▼
Event Grid
          │
          ▼
Storage Queue
(blob-sync-events)
          │
          ▼
Azure VM Listener
(blob-to-gcs)
          │
          ▼
PostgreSQL
(blob_events)
          │
          ▼
Google Cloud Storage
(highradius-poc-gcs-blobstorage)

Show more lines
Once the Event Subscription is created successfully, the next step is:




Plain Text
Build Azure VM Listener
and I'll give you:

PostgreSQL blob_events table
.env
Full listener.py
Complete systemd service
Validation procedure
for the rollback architecture.





You said:

Copilot said:
This screen looks correct. ✅

For your enterprise rollback architecture, I would use the following settings:

Endpoint Details



Plain Text
Endpoint Type:
Storage Queue ✅
 
Queue:
blob-sync-events ✅
Cross-Tenant Delivery



Plain Text
Disabled ✅
Keep it disabled unless the storage account and queue are in different Azure tenants.

Managed Identity for Delivery
Current:




Plain Text
Managed Identity Type:
None
For your design:




Plain Text
Azure Blob
    ↓
Event Grid
    ↓
Storage Queue
    ↓
Azure VM Listener
Keep it as:




Plain Text
None ✅
Because Event Grid can write directly to the Storage Queue in the same subscription without requiring a managed identity.

Event Types Selected
You mentioned the client wants all six:




Plain Text
✅ Blob Created
✅ Blob Deleted
✅ Directory Created
✅ Directory Deleted
✅ Blob Renamed
✅ Directory Renamed
That's acceptable, provided the listener handles each event type appropriately.

Before Clicking Create
Verify:




Plain Text
Name:
blob-to-gcs-eventgrid
 
Event Schema:
Event Grid Schema
 
Events:
6 selected
 
Endpoint Type:
Storage Queue
 
Queue:
blob-sync-events
 
Managed Identity:
None
 
Dead Lettering:
Enabled
 
Retry Policy:
30 attempts
 
TTL:
1440 minutes
 
Expiration:
Disabled

Show more lines
What Happens Next
After you click Create:




Plain Text
Azure Blob
       ↓
Event Grid
       ↓
blob-sync-events Queue
The next implementation step will be:

Build Queue Consumer
We'll create:




Plain Text
/opt/blob-to-gcs
and then:




Plain Text
listener.py
that:

Polls Azure Storage Queue
Reads Event Grid messages
Handles all 6 event types
Writes audit records to PostgreSQL (blob_events)
Uploads files to GCS
Deletes files from GCS when needed
Runs as a systemd service
That listener implementation is the next major deliverable after this Event Grid subscription is created successfully.





You said:

still create option not highlight 
Copilot said:
The Create button is still disabled because one of the required fields on another tab is not valid or not completed.

From your screenshot, the most likely causes are:

1. System Topic Name (Most Common)
Scroll back to the Basics tab and check:




Plain Text
System Topic Name *
Make sure it contains a value, for example:




Plain Text
blob-sync-system-topic
or




Plain Text
hirteststoragemigration-events
This field is mandatory.

2. Filters Tab
Since you selected all 6 event types, go to:




Plain Text
Filters
Verify there are no validation errors.

Recommended:




Plain Text
Subject Begins With:
/blobServices/default/containers/testcontainer/
If a filter value is partially entered or invalid, Azure disables Create.

3. Additional Features Tab
You enabled:




Plain Text
Configure Retry Policies
Check the Event Time To Live fields.

The screenshot earlier showed:




Plain Text
Days    = 0
Hours   = (blank)
Minutes = 1440
Seconds = 0
This can cause validation issues.

Set:




Plain Text
Days    = 1
Hours   = 0
Minutes = 0
Seconds = 0
Or:




Plain Text
Days    = 0
Hours   = 24
Minutes = 0
Seconds = 0
depending on what Azure accepts in your portal version.

4. Expiration Time
Make sure:




Plain Text
Enable expiration time
is unchecked.

Production subscriptions should not expire automatically.

5. Endpoint Validation
Confirm:




Plain Text
Endpoint Type:
Storage Queue
 
Endpoint:
blob-sync-events
If the queue was recently created, click:




Plain Text
change
and reselect the queue.

Quick Check
Go back through the tabs in this order:




Plain Text
Basics
 ↓
Filters
 ↓
Additional Features
 ↓
Delivery Properties
Look for any red asterisks (*) or red validation messages.

The two fields most likely blocking Create are:




Plain Text
System Topic Name
or




Plain Text
Event Time To Live
Tell me:




Plain Text
What value is in System Topic Name?
and whether Enable expiration time is currently checked or unchecked, and I'll pinpoint the exact issue.





You said:

Copilot said:
The Basics tab looks valid.

The reason Create is still disabled is almost certainly on one of the other tabs:

Filters
Additional Features
Delivery Properties
Based on your earlier screenshot, the most likely issue is still in Additional Features.

Check 1: Event Subscription Expiration Time
Go to Additional Features and make sure:




Plain Text
Enable expiration time
is unchecked.

For production:




Plain Text
❌ Disable expiration time
If it remains enabled, Azure may require a valid future timestamp and can block creation.

Check 2: Event Time To Live
If Retry Policies are enabled, verify the TTL fields are valid.

Recommended:




Plain Text
Maximum Delivery Attempts: 30



Plain Text
Days: 1
Hours: 0
Minutes: 0
Seconds: 0
Avoid:




Plain Text
Days: 0
Minutes: 1440
Some portal versions accept it, some trigger validation issues.

Check 3: Filters Tab
Open Filters.

Verify:




Plain Text
Subject Begins With
contains a valid path, for example:




Plain Text
/blobServices/default/containers/testcontainer/
or leave it completely blank.

Do not leave a partially entered value.

Check 4: Storage Queue Permissions
Sometimes Event Grid cannot validate the queue endpoint if queue permissions aren't resolved yet.

Click:




Plain Text
change
next to:




Plain Text
blob-sync-events
and reselect:




Plain Text
Storage Account:
hirteststoragemigration
 
Queue:
blob-sync-events
Quickest Way to Find the Error
Go through the tabs:




Plain Text
Basics
Filters
Additional Features
Delivery Properties
and look for:

Red text
Red borders
Red asterisks with warnings
Azure usually marks the exact field preventing creation.

Send Screenshots
Please send screenshots of:

Filters
Additional Features
Delivery Properties
and I'll pinpoint exactly what's blocking the Create button.





You said:
Dead-Lettering
Save events that cannot be delivered to storage. Learn more 
Enable dead-lettering
Cross-tenant delivery
Disabled
Storage Subscription
HiRAzurePOC2026
Storage Blob Container
eventgrid-deadletter
 (change) 
Managed Identity for Dead-Letter
Managed identity type
None
Retry Policies
Customize how many times and for how long event delivery will be retried. Learn more 
Configure Retry Policies
Events will be dead-lettered or discarded after either the number of delivery attempts is exceeded or the time to live has elapsed.
Max Event Delivery Attempts
Event Time to Live
Days
Hours
Minutes
Seconds
Event Subscription Expiration Time
Set a time at which the event subscription will automatically be deleted.
Enable expiration time
Labels
Set metadata on this event subscription that is only visible through this experience. The metadata (labels) set does not affect the data plane behavior in any way. For example, labels do not show anywhere in the request or in the events that are sent to subscribers.



Copilot said:
I found the issue. ✅

The Create button is disabled because under Retry Policies, you've enabled:




Plain Text
Configure Retry Policies
but you haven't filled the required values:




Plain Text
Max Event Delivery Attempts
Event Time to Live
Azure treats these as mandatory when retry policies are enabled.

Recommended Values
Max Event Delivery Attempts



Plain Text
30
Event Time To Live
Set:




Plain Text
Days    = 1
Hours   = 0
Minutes = 0
Seconds = 0
This gives:




Plain Text
24 Hours
which is a good production value.

Event Subscription Expiration Time
Make sure:




Plain Text
Enable expiration time
is:




Plain Text
❌ Unchecked
Do not enable expiration in production.

Dead Lettering
Current values are good:




Plain Text
✅ Enable dead-lettering
 
Container:
eventgrid-deadletter
 
Managed Identity:
None
Labels (Optional)
You can add:




Plain Text
Application = HighRadius
Environment = Production
Component = BlobToGCSRollback
but these are optional and do not affect functionality.

Final Additional Features Configuration



Plain Text
Enable dead-lettering          = Yes
Dead Letter Container          = eventgrid-deadletter
 
Configure Retry Policies       = Yes
 
Max Delivery Attempts          = 30
 
Event Time To Live
Days                           = 1
Hours                          = 0
Minutes                        = 0
Seconds                        = 0
 
Enable Expiration Time         = No
After entering the retry values, the Create button should become active.

Once the Event Grid Subscription is created successfully, we'll move to:




Plain Text
Step 3: Create GCP Service Account
 
azure-to-gcs-sa
and continue building the rollback path:




Plain Text
Azure Blob
 ↓
Event Grid
 ↓
Storage Queue
 ↓
Listener VM
 ↓
PostgreSQL
 ↓
Google Cloud Storage







Message Copilot


AI-generated content may be incorrect
https://copilot.cloud.microsoft/chat/conversation/0f536a19-348d-47a1-b6b5-02c9bd8c9185?fromCode=cmcv2&redirectId=788F9C6047654C1A991550D8E35B3042

BLOB BLOB BLOB BLOB



You said:
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$ journalctl -u blob-to-gcs -n 100 --no-pager
-- No entries --
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$
Copilot said:
That output is actually very revealing:




Plain Text
journalctl -u blob-to-gcs -n 100 --no-pager
 
-- No entries --
What it means
Your service is not running under systemd yet, or the service has never been started successfully.

So right now there are two different things:

✅ Queue contains the BlobDeleted event (Messages: 1)
❌ There are no blob-to-gcs service logs
That means we should stop looking at the Event Grid side for now and verify the service.

Check if the service exists
Run:




Shell
sudo systemctl status blob-to-gcs
Paste the output.

Check if the service file exists
Run:




Shell
ls -l /etc/systemd/system/blob-to-gcs.service
If service exists, reload and start it



Shell
sudo systemctl daemon-reload
 
sudo systemctl enable blob-to-gcs
 
sudo systemctl start blob-to-gcs
Then:




Shell
sudo systemctl status blob-to-gcs
If service is running, view logs



Shell
journalctl -u blob-to-gcs -f
At that point we should see messages such as:




Plain Text
Starting Azure Blob -> GCS listener
or an error/traceback.

Important Finding So Far
We already proved:

✅ rollback.txt delete event reached the queue




JSON
"eventType":"Microsoft.Storage.BlobDeleted"
✅ Queue has pending messages




Plain Text
Messages: 1
✅ PostgreSQL did not receive a BlobDeleted record

Therefore the event is sitting unprocessed.

That strongly suggests either:




Plain Text
blob_to_gcs_listener.py is not running
or




Plain Text
blob_to_gcs_listener.py is crashing before processing the delete event
The next command that will tell us exactly where we are is:




Shell
sudo systemctl status blob-to-gcs
Please run that and paste the output.





You said:
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$ journalctl -u blob-to-gcs -n 100 --no-pager
-- No entries --
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$ sudo systemctl status blob-to-gcs
○ blob-to-gcs.service - Azure Blob To GCS Sync
     Loaded: loaded (/etc/systemd/system/blob-to-gcs.service; disabled; preset: enabled)
     Active: inactive (dead)
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$ sudo systemctl start blob-to-gcs
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$ sudo systemctl status blob-to-gcs
● blob-to-gcs.service - Azure Blob To GCS Sync
     Loaded: loaded (/etc/systemd/system/blob-to-gcs.service; disabled; preset: enabled)
     Active: active (running) since Fri 2026-09-18 10:00:16 UTC; 1s ago
   Main PID: 23528 (python)
      Tasks: 9 (limit: 4598)
     Memory: 50.2M (peak: 50.5M)
        CPU: 908ms
     CGroup: /system.slice/blob-to-gcs.service
             └─23528 /opt/blob-to-gcs/venv/bin/python /opt/blob-to-gcs/blob_to_gcs_listener.py


Sep 18 10:00:17 gcstoblobstorage python[23528]: 2026-09-18 10:00:17,991 INFO ThreadPoolExecutor-0_2 Response status: 200
Sep 18 10:00:17 gcstoblobstorage python[23528]: Response headers:
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'Cache-Control': 'no-cache'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'Transfer-Encoding': 'chunked'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'Content-Type': 'application/xml'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'Server': 'Windows-Azure-Queue/1.0 Microsoft-HTTPAPI/2.0'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'x-ms-request-id': '61c11063-b003-0085-5854-475e77000000'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'x-ms-client-request-id': 'c08f985e-b347-11f1-8cdb-e4fb1eff1455'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'x-ms-version': 'REDACTED'
Sep 18 10:00:17 gcstoblobstorage python[23528]:     'Date': 'Fri, 18 Sep 2026 10:00:17 GMT'
(venv) trianzadmin@gcstoblobstorage:/opt/blob-to-gcs$

Copilot said:
Excellent. ✅ We now know exactly why the delete wasn't working before.

What happened
Before:




Plain Text
blob-to-gcs service = Inactive (dead)
So:




Plain Text
BlobCreated events
    → were processed manually
 
BlobDeleted events
    → stayed in the queue
Now:




Plain Text
blob-to-gcs service = Active (running)
and the queue still had:




Plain Text
Messages: 1
which was your rollback.txt delete event.

Next thing to check
Wait 30-60 seconds and run:




SQL
SELECT
    blob_name,
    event_type,
    status,
    error_message,
    created_at
FROM blob_events
ORDER BY created_at DESC
LIMIT 20;
I'm expecting a new row like:




Plain Text
rollback.txt | Microsoft.Storage.BlobDeleted | COMPLETED
If you see that row, then the listener consumed the Event Grid delete message successfully.

Check GCS
Run:




Shell
gcloud storage ls gs://highradius-poc-gcs-blobstorage/rollback.txt
Expected if everything worked



Plain Text
Listed 0 items.
or




Plain Text
Object not found
If file still exists
Then we'll inspect:




Shell
journalctl -u blob-to-gcs -n 100 --no-pager
and specifically look for:




Plain Text
BlobDeleted
FAILED
or a traceback.

Also verify queue is empty now
Run:




Shell
python /tmp/check_queue.py
Expected:




Plain Text
Messages: 0
If it becomes 0, we know the delete message was consumed.

Summary
Current status:

✅ Azure Blob → Event Grid works
✅ Event Grid → Queue works
✅ Queue contains delete event
✅ blob-to-gcs service is now running
✅ PostgreSQL is working
✅ GCS upload works

Now we only need to verify:




Plain Text
BlobDeleted
     ↓
Consumed from Queue
     ↓
PostgreSQL Row
     ↓
GCS File Removed
Run these three checks:




SQL
SELECT
    blob_name,
    event_type,
    status
FROM blob_events
ORDER BY created_at DESC
LIMIT 20;



Shell
gcloud storage ls gs://highradius-poc-gcs-blobstorage/rollback.txt



Shell
python /tmp/check_queue.py
and share the outputs. That will confirm whether the rollback delete path is fully working. 🚀





You said:
some 1kb files up to 20 mb uploaded in azure blob storage can we check flow
Copilot said:
Yes ✅. Since your reverse listener and PostgreSQL are working, the easiest way to validate the full flow is to upload a batch of files (1 KB → 20 MB) into Azure Blob and then track them through all layers.

Validation Flow



Plain Text
Azure Blob
    ↓
Event Grid
    ↓
Storage Queue
(blob-sync-events)
    ↓
blob_to_gcs_listener.py
    ↓
PostgreSQL
(blob_events)
    ↓
GCS
(highradius-poc-gcs-blobstorage)
1. Upload Test Files
Upload a mix of files:




Plain Text
1KB_1.txt
1KB_2.txt
10MB_1.bin
20MB_1.bin
into:




Plain Text
Storage Account:
hirteststoragemigration
 
Container:
testcontainer
2. Check Queue Growth
Run on VM:




Shell
python /tmp/check_queue.py
Expected initially:




Plain Text
Messages: 5
Messages: 10
Messages: 20
depending on uploads.

After processing:




Plain Text
Messages: 0
3. Monitor Listener



Shell
journalctl -u blob-to-gcs -f
Expected:




Plain Text
BlobCreated
 
SOURCE=...
 
COMPLETED
For larger files:




Plain Text
1MB
5MB
10MB
20MB
you should see longer processing times.

4. Check PostgreSQL
Connect:




Shell
sudo -u postgres psql -d gcssync
Check totals:




SQL
SELECT
    status,
    count(*)
FROM blob_events
GROUP BY status;
Expected:




Plain Text
COMPLETED
Check recent files:




SQL
SELECT
    blob_name,
    event_type,
    status,
    created_at
FROM blob_events
ORDER BY created_at DESC
LIMIT 50;
Expected:




Plain Text
1KB_1.txt
10MB_1.bin
20MB_1.bin
 
Microsoft.Storage.BlobCreated
 
COMPLETED
5. Verify GCS Bucket
Count files:




Shell
gcloud storage ls \
gs://highradius-poc-gcs-blobstorage/** | wc -l
Check specific file:




Shell
gcloud storage ls \
gs://highradius-poc-gcs-blobstorage/20MB_1.bin
Expected:




Plain Text
gs://highradius-poc-gcs-blobstorage/20MB_1.bin
6. Verify Integrity
Download file from GCS:




Shell
gcloud storage cp \
gs://highradius-poc-gcs-blobstorage/20MB_1.bin .
Check size:




Shell
ls -lh 20MB_1.bin
Compare with Azure source file size.

7. Throughput Validation
Monitor VM:




Shell
top
or:




Shell
htop
Check:




Plain Text
CPU
Memory
Load Average
For your VM:




Plain Text
Standard_B2als_v2
Current recommendation:




INI
WORKERS=8
If processing remains smooth:

✅ keep WORKERS=8

If queue grows faster than consumption:

✅ increase VM size ✅ later increase WORKERS

Success Criteria
You can consider rollback replication validated when:




Plain Text
✅ File uploaded to Blob
✅ Queue receives event
✅ Listener processes event
✅ PostgreSQL row created
✅ Status = COMPLETED
✅ File exists in GCS
For delete validation:




Plain Text
✅ File deleted in Blob
✅ BlobDeleted event received
✅ PostgreSQL row created
✅ File removed from GCS
I would test:




Plain Text
10 x 1KB files
10 x 1MB files
5 x 10MB files
2 x 20MB files
first, verify PostgreSQL shows COMPLETED for all, then move to larger bulk-volume testing.





You said:

Copilot said:
This screenshot is actually very encouraging. ✅

What the screenshot proves
You copied 1008 files to Azure Blob:




Plain Text
Transferred: 1008 / 1008, 100%
Elapsed time: 32.0s
Total data:




Plain Text
19.573 MiB
Average throughput:




Plain Text
1.145 MiB/s
Files transferred include:




Plain Text
testfile_980.txt
testfile_981.txt
testfile_982.txt
...
testfile_999.txt
What should happen now?
Those uploads should generate:




Plain Text
BlobCreated
events.

Flow:




Plain Text
Azure Blob
        ↓
Event Grid
        ↓
Storage Queue
(blob-sync-events)
        ↓
blob_to_gcs_listener.py
        ↓
PostgreSQL
(blob_events)
        ↓
Google Cloud Storage
Validation 1
Check queue size:




Shell
python /tmp/check_queue.py
Expected:




Plain Text
Messages: 1008
or some number that gradually decreases while the listener processes files.

Validation 2
Check Listener Service




Shell
sudo systemctl status blob-to-gcs
Expected:




Plain Text
active (running)
Validation 3
Watch processing




Shell
journalctl -u blob-to-gcs -f
Expected:




Plain Text
BlobCreated
Processing testfile_980.txt
COMPLETED
Validation 4
Check PostgreSQL




Shell
sudo -u postgres psql -d gcssync



SQL
SELECT
    count(*)
FROM blob_events;
Expected:




Plain Text
1008+
Check completed:




SQL
SELECT
    status,
    count(*)
FROM blob_events
GROUP BY status;
Expected:




Plain Text
COMPLETED
Validation 5
Check GCS

From Cloud Shell:




Shell
gcloud storage ls gs://highradius-poc-gcs-blobstorage/ | grep testfile_
Or count:




Shell
gcloud storage ls gs://highradius-poc-gcs-blobstorage/** | wc -l
What I Would Measure
PostgreSQL



SQL
SELECT
    count(*)
FROM blob_events
WHERE status='COMPLETED';
Queue Backlog



Shell
python /tmp/check_queue.py
GCS Count



Shell
gcloud storage ls gs://highradius-poc-gcs-blobstorage/** | wc -l
Success Criteria
For your 1008 file test:




Plain Text
Uploaded to Azure Blob     = 1008
Queue Events               = 1008
PostgreSQL Completed       = 1008
Files in GCS               = 1008
If those numbers match, you've proven:

✅ High-volume upload works
✅ Event Grid scales
✅ Queue scales
✅ PostgreSQL audit tracking works
✅ Azure Blob → GCS rollback path works
✅ Enterprise rollback architecture is functioning end-to-end

This is exactly the kind of bulk test I would run during Hypercare validation. 🚀







How to automate the full validation?

What if some files are missing in GCS?

How to verify each file individually?



✅ Blob Created
2
✅ Blob Deleted
3
✅ Directory Created
4
✅ Directory Deleted
5
✅ Blob Renamed
6
✅ Directory Renamed




