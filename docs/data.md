## Adding some data to our sandbox

First we fill the condor_history withg some toy data. Therefore we need to install htcondor

### Install htcondor

```
sudo dnf config-manager --set-enabled crb
yum install -y https://research.cs.wisc.edu/htcondor/repo/24.x/htcondor-release-current.el9.noarch.rpm
yum install -y condor
```

#### minimal configuration

add all daemons to the config file:
/etc/condor/condor_config.local
```
DAEMON_LIST = MASTER, SCHEDD, COLLECTOR, STARTD

ALLOW_READ = *
ALLOW_WRITE = *

NETWORK_INTERFACE = 0.0.0.0
```
add the IP-Address of you test VM and the proper port to the common config file

/etc/condor/config.d/00-common.conf 
```
CONDOR_HOST = <replace with your IP adress>
COLLECTOR_PORT = 9618
```
start condor with:
```
 sudo systemctl start condor
 sudo systemctl enable condor 
```
check if condor is running:
```
 condor_status
```
if not, restart with `condor_restart` and check again.


### Install pyauditor and other modules - required to create the mock data

Now we can download this git repo with the mock_history_insertion.py script.
You might need to install git via dnf.
```
git clone https://github.com/ALU-Schumacher/auditor-demo.git
```

Create a python venv e.g. in the user home dir (python might be called python3):
```
cd 
python -m venv .venv
source .venv/bin/activate
cd auditor-demo
```
Then you can install the requirements from the requirements.txt of this repo
```
pip install -r requirements.txt
```
Now we can create the pseudo-condor data:
```
python mock_history_insertion.py
```
afterwards we can deactivate the env again:
```
deactivate
```

We can check if that was successfull with the `condor_history` command:
```
condor_history 
 ID     OWNER          SUBMITTED   RUN_TIME     ST COMPLETED   CMD            
999.0   user-ilc-005    3/10 03:48   0+03:00:00 C   5/1  19:38 /ce_home/arc/sessiondir/testJob_000000999/condorjob.sh 
998.0   user-cms-001    3/10 03:48   0+18:00:00 C   5/2  10:37 /ce_home/arc/sessiondir/testJob_000000998/condorjob.sh 
997.0   user-ilc-002    3/10 03:48   0+08:00:00 C   5/2  00:36 /ce_home/arc/sessiondir/testJob_000000997/condorjob.sh 
996.0   user-atlas-006  3/10 03:48   0+08:00:00 C   5/2  00:35 /ce_home/arc/sessiondir/testJob_000000996/condorjob.sh 
995.0   user-cms-005    3/10 03:48   0+10:00:00 C   5/2  02:34 /ce_home/arc/sessiondir/testJob_000000995/condorjob.sh 
994.0   user-cms-008    3/10 03:48   0+07:00:00 C   5/1  23:33 /ce_home/arc/sessiondir/testJob_000000994/condorjob.sh 
993.0   user-ilc-006    3/10 03:48   0+16:00:00 C   5/2  08:32 /ce_home/arc/sessiondir/testJob_000000993/condorjob.sh 
...
```
Nice! Data is in condor. Now we can run the condor-collector (or wait until the service comes around).

### Run AUDITOR-HTCondor collector 

Here we are impatient, we  want to execute the collector manually.
Therefore we can call:
```
/opt/auditor_htcondor_collector/venv/bin/python /opt/auditor_htcondor_collector/venv/bin/auditor-htcondor-collector --config /etc/auditor/auditor_htcondor_collector.yml --job-id  0.0
```
you should see something like:
```
/opt/auditor_htcondor_collector/venv/bin/python /opt/auditor_htcondor_collector/venv/bin/auditor-htcondor-collector --config /etc/auditor/auditor_htcondor_collector.yml --job-id  0.0


2025-07-07 13:39:42,890 - auditor.collectors.htcondor - INFO     - Using AUDITOR client at localhost:8000.
2025-07-07 13:39:42,890 - auditor.collectors.htcondor - INFO     - Using timeout of 10 seconds for AUDITOR client.
2025-07-07 13:39:42,892 - auditor.collectors.htcondor - INFO     - Starting collector run.
2025-07-07 13:39:42,892 - auditor.collectors.htcondor - INFO     - Collecting jobs for schedd 'auditor-demo.novalocal'.
2025-07-07 13:39:49,813 - auditor.collectors.htcondor - INFO     - Added 999 records.
2025-07-07 13:39:49,816 - auditor.collectors.htcondor - INFO     - Collector run finished.

```
Just interrupt the command with `Strg+c`.

In order to test the entire pipeline, you can run the APEL-plugin in dry-run mode:

```
 /opt/auditor_apel_plugin/venv/bin/python /opt/auditor_apel_plugin/venv/bin/auditor-apel-publish --config /etc/auditor/auditor_apel_plugin.yml --dry-run
```
#### Wipe the DB and re-un the AUDITOR-HTCondor collector 

If we want to wipe the database for another run of the collector, we need to execute the following commands

```
psql -h localhost -U postgres -d auditor

auditor=# DELETE FROM auditor_accounting;
```
and remove the check point of the collector:
```
rm /opt/auditor_htcondor_collector/htcondor_history_state.db 
```
The we can execute the collector command again.


### Fill AUDITOR with toy data

If we want to skip the part of installing HTCondor and we just want to mock the APEL plugin step, we can use the other mocking skript: `mock_records_insertion.py`
Wipe the DB as described above and execute the `mock_records_insertion.py ` in our venv:

Now you can execute the mock_records_insertion.py script, which adds 1k records with random data into your AUDITOR db.
The script is available in this repo. execute the following command in the directory where you have downloaded this repo:
```
 python mock_records_insertion.py
```
If you now run the auditor-apel-publish command with the --dry-run option:
```
/opt/auditor_apel_plugin/venv/bin/python /opt/auditor_apel_plugin/venv/bin/auditor-apel-publish --config /etc/auditor/auditor_apel_plugin.yml  --dry-run
```

You should get a summary output as follows:
```
[2025-04-30 15:31:21] INFO     Starting one-shot dry-run, nothing will be sent to APEL! (/opt/auditor_apel_plugin/venv/lib/python3.9/site-packages/auditor_apel_plugin/publish.py at line 47)
[2025-04-30 15:31:21] INFO     Getting records for site SITE_A with site_ids: ['site_id_1', 'site_id_2'] (/opt/auditor_apel_plugin/venv/lib/python3.9/site-packages/auditor_apel_plugin/core.py at line 45)
[2025-04-30 15:31:22] INFO     Total numbers reported by the plugin:
Site: SITE_A
Month: 5
Year: 2025
NumberOfJobs: 6731
WallDuration: 276951600
NormalisedWallDuration: 5013237600
CpuDuration: 274186957
NormalisedCpuDuration: 4963289834

 (/opt/auditor_apel_plugin/venv/lib/python3.9/site-packages/auditor_apel_plugin/publish.py at line 134)
[2025-04-30 15:31:22] INFO     Getting records for site SITE_B with site_ids: ['site_id_3'] (/opt/auditor_apel_plugin/venv/lib/python3.9/site-packages/auditor_apel_plugin/core.py at line 45)
[2025-04-30 15:31:22] INFO     Total numbers reported by the plugin:
Site: SITE_B
Month: 5
Year: 2025
NumberOfJobs: 3269
WallDuration: 134074800
NormalisedWallDuration: 2406434400
CpuDuration: 132726130
NormalisedCpuDuration: 2382405970

```

## Accessing Data with python-auditor
### Access Data in ipython Session

Start ipython - you might need to install it with dnf:
```
ipython
```
Import required modules
```
import numpy as np
import datetime
import json
from pyauditor import AuditorClientBuilder, Value, Operator, QueryBuilder
```
Connect to AUDITOR

```
builder = AuditorClientBuilder()
builder = builder.address("127.0.0.1", 8000)
client = builder.build()
```
Create a proper query using the pyauditor QueryBuilder:
```
now = datetime.datetime.now()
start = datetime.datetime(now.year,now.month , 1, tzinfo=datetime.timezone.utc)
# Set the datetime value in Utc using Value object
value = Value.set_datetime(start)
query_string = QueryBuilder().with_stop_time(Operator().gte(value)).build()
query_string
```
```
Out[3]: 'stop_time[gte]=2025-05-01T00%3A00%3A00%2B00%3A00'
```
Execute the query:
```
records = await client.advanced_query(query_string)
```
Now you can have a look at the data (first 10 records):
```
records[:10]
```

```
Out[5]: 
[Record { record_id: "record-13", meta: Some(Meta({"group_id": ["group_1"], "site_id": ["site_3"], "user_id": ["user-13"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(16), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(3072), scores: [] }]), start_time: Some(2025-05-01T00:13:00Z), stop_time: Some(2025-05-01T01:13:00Z), runtime: Some(3600) },
 Record { record_id: "record-49", meta: Some(Meta({"group_id": ["group_3"], "user_id": ["user-49"], "site_id": ["site_2"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(4), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(4096), scores: [] }]), start_time: Some(2025-05-01T00:49:00Z), stop_time: Some(2025-05-01T01:49:00Z), runtime: Some(3600) },
 Record { record_id: "record-60", meta: Some(Meta({"group_id": ["group_3"], "site_id": ["site_2"], "user_id": ["user-60"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(5), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(2048), scores: [] }]), start_time: Some(2025-05-01T01:00:00Z), stop_time: Some(2025-05-01T02:00:00Z), runtime: Some(3600) },
 Record { record_id: "record-20", meta: Some(Meta({"group_id": ["group_1"], "user_id": ["user-20"], "site_id": ["site_1"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(7), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(6144), scores: [] }]), start_time: Some(2025-05-01T00:20:00Z), stop_time: Some(2025-05-01T02:20:00Z), runtime: Some(7200) },
 Record { record_id: "record-48", meta: Some(Meta({"site_id": ["site_1"], "group_id": ["group_1"], "user_id": ["user-48"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(10), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(4096), scores: [] }]), start_time: Some(2025-05-01T00:48:00Z), stop_time: Some(2025-05-01T02:48:00Z), runtime: Some(7200) },
 Record { record_id: "record-3", meta: Some(Meta({"group_id": ["group_3"], "site_id": ["site_3"], "user_id": ["user-3"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(15), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(6144), scores: [] }]), start_time: Some(2025-05-01T00:03:00Z), stop_time: Some(2025-05-01T03:03:00Z), runtime: Some(10800) },
 Record { record_id: "record-9", meta: Some(Meta({"user_id": ["user-9"], "group_id": ["group_3"], "site_id": ["site_2"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(8), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(4096), scores: [] }]), start_time: Some(2025-05-01T00:09:00Z), stop_time: Some(2025-05-01T03:09:00Z), runtime: Some(10800) },
 Record { record_id: "record-74", meta: Some(Meta({"group_id": ["group_1"], "site_id": ["site_3"], "user_id": ["user-74"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(6), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(2048), scores: [] }]), start_time: Some(2025-05-01T01:14:00Z), stop_time: Some(2025-05-01T03:14:00Z), runtime: Some(7200) },
 Record { record_id: "record-39", meta: Some(Meta({"user_id": ["user-39"], "site_id": ["site_3"], "group_id": ["group_1"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(9), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(3072), scores: [] }]), start_time: Some(2025-05-01T00:39:00Z), stop_time: Some(2025-05-01T03:39:00Z), runtime: Some(10800) },
 Record { record_id: "record-162", meta: Some(Meta({"site_id": ["site_1"], "user_id": ["user-162"], "group_id": ["group_3"]})), components: Some([Component { name: ValidName("cpu"), amount: ValidAmount(14), scores: [Score { name: ValidName("hepspec23"), value: ValidValue(10.0) }] }, Component { name: ValidName("mem"), amount: ValidAmount(7168), scores: [] }]), start_time: Some(2025-05-01T02:42:00Z), stop_time: Some(2025-05-01T03:42:00Z), runtime: Some(3600) }]
```
If you want to use the data for further analysis, you can transform the records it a json-object
```
# transform record into json
json.loads(records[0].to_json())
```
```
Out[6]: 
{'components': [{'amount': 16,
   'name': 'cpu',
   'scores': [{'name': 'hepspec23', 'value': 10.0}]},
  {'amount': 3072, 'name': 'mem', 'scores': []}],
 'meta': {'group_id': ['group_1'],
  'site_id': ['site_3'],
  'user_id': ['user-13']},
 'record_id': 'record-13',
 'runtime': 3600,
 'start_time': '2025-05-01T00:13:00Z',
 'stop_time': '2025-05-01T01:13:00Z'}
```
```
print(json.dumps(json.loads(records[13].to_json()),indent=3))
```
```
{
   "components": [
      {
         "amount": 14,
         "name": "cpu",
         "scores": [
            {
               "name": "hepspec23",
               "value": 10.0
            }
         ]
      },
      {
         "amount": 2048,
         "name": "mem",
         "scores": []
      }
   ],
   "meta": {
      "group_id": [
         "group_2"
      ],
      "site_id": [
         "site_3"
      ],
      "user_id": [
         "user-68"
      ]
   },
   "record_id": "record-68",
   "runtime": 10800,
   "start_time": "2025-05-01T01:08:00Z",
   "stop_time": "2025-05-01T04:08:00Z"
}
```
