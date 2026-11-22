FLEET_JSON=$(curl -sg "http://localhost:8080/computer/api/json?tree=computer[displayName,offline,numExecutors,idle,assignedLabels[name]]")
echo "${FLEET_JSON}"




curl -s -X POST "http://localhost:8080/scriptText" \
  --data-urlencode 'script=
import jenkins.model.Jenkins
import hudson.slaves.DumbSlave
import hudson.slaves.JNLPLauncher
import hudson.model.Node

def j = Jenkins.get()
def nodeName = "edge-worker-01"

if (j.getNode(nodeName) == null) {
    def agent = new DumbSlave(
        nodeName,
        "/tmp/jenkins-agent-01",
        new JNLPLauncher(true)
    )
    agent.setNumExecutors(2)
    agent.setMode(Node.Mode.EXCLUSIVE)
    agent.setLabelString("linux-edge fast-io")
    j.addNode(agent)
    println "CREATED_AGENT: ${nodeName}"
} else {
    println "EXISTS_AGENT: ${nodeName}"
}
'



QUEUE_JSON=$(curl -sg "http://localhost:8080/queue/api/json?tree=items[id,why,task[name]]")
QUEUE_DEPTH=$(echo "${QUEUE_JSON}" | grep -o '"id":[0-9]*' | wc -l | tr -d ' ')

echo "Current Queue Depth: ${QUEUE_DEPTH}"
if [ "${QUEUE_DEPTH}" -ge 3 ]; then
    echo "[BACKPRESSURE] Queue saturated (${QUEUE_DEPTH} items waiting). Throttling dispatch."
    exit 1
fi



NODE_NAME="edge-worker-01"
REASON="Automated%20quarantine%3A%20high%20latency%20detected"

# Toggle offline state with an audit message
curl -s -X POST "http://localhost:8080/computer/${NODE_NAME}/toggleOffline?offlineMessage=${REASON}"

# Verify offline status and message via REST
curl -sg "http://localhost:8080/computer/${NODE_NAME}/api/json?tree=displayName,offline,offlineCauseReason"




curl -s -X POST "http://localhost:8080/scriptText" \
  --data-urlencode 'script=
import jenkins.model.Jenkins
import hudson.model.FreeStyleProject

def j = Jenkins.get()
def job = j.getItem("lesson9-audit-worker") ?: j.createProject(FreeStyleProject, "lesson9-audit-worker")

job.setAssignedLabel(j.getLabel("master"))
job.save()
println "BOUND_LABEL: ${job.getAssignedLabelString()}"
'




# 1. Trigger a fresh queued item
QUEUE_URL=$(curl -s -D - -o /dev/null -X POST \
  "http://localhost:8080/job/queue-cancel-test/build?delay=0sec" \
  | grep -i "^Location:" | awk '{print $2}' | tr -d '\r\n')

# 2. Extract the numeric Queue ID
QUEUE_ID=$(echo "${QUEUE_URL}" | sed -n 's#.*/queue/item/\([0-9]*\)/#\1#p')
echo "Live Queue ID: ${QUEUE_ID}"

# 3. Pass id in the POST form body (-d)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
  -d "id=${QUEUE_ID}" \
  "http://localhost:8080/queue/cancelItem")
echo "Cancel HTTP Status: ${HTTP_CODE}"




curl -s -X POST "http://localhost:8080/scriptText" \
  --data-urlencode 'script=
import jenkins.model.Jenkins

def j = Jenkins.get()

// 1. Wipe workspace on an idle job safely
def job = j.getItem("fleet-routing-worker")
if (job != null && !job.isBuilding()) {
    job.doDoWipeOutWorkspace()
    println "WIPED_WORKSPACE: ${job.name}"
}

// 2. Remove an ephemeral agent only if it is idle
def node = j.getNode("edge-worker-01")
if (node != null && node.toComputer()?.isIdle()) {
    j.removeNode(node)
    println "REMOVED_AGENT: edge-worker-01"
}
'


****************************************



Practice Test 1 : Ephemeral Agent Lifecycle, Idempotent Cordon & Fleet Cleanup

Specification
Write a standalone script (pt1_fleet_lifecycle.sh) that manages the full cradle-to-grave lifecycle of an ephemeral worker node without manual UI intervention:
1.	Idempotent Node Registration: Register a virtual agent named pt1-ephemeral-node via /scriptText with numExecutors=3, Node.Mode.EXCLUSIVE, and label "tier-gold eu-west". Run the registration step twice in your script to prove it handles existing nodes cleanly without errors.
2.	REST Fleet Audit: Query /computer/pt1-ephemeral-node/api/json using curl -sg and ?tree= to extract and print:
o	displayName
o	numExecutors
o	assignedLabels
3.	Idempotent Quarantine (Cordon): Check temporarilyOffline via REST; if false, cordon pt1-ephemeral-node via POST /computer/pt1-ephemeral-node/toggleOffline with offlineMessage=PT1%20Audit%20Lock, and assert via REST that "temporarilyOffline":true and "offlineCauseReason" contains PT1 Audit Lock.
4.	Graceful Decommissioning: Remove pt1-ephemeral-node via Groovy (j.removeNode(node)) after verifying isIdle(), and confirm via HTTP status code (-w "%{http_code}") that /computer/pt1-ephemeral-node/api/json now returns HTTP 404.



Practice Test 2: Stalled-Queue Detection, Self-Healing Failover & Queue Cancellation

Specification
Write a standalone script (pt2_queue_failover.sh) that proves your controller can detect a stalled workload targeting an offline node label, dynamically reroute it to a healthy node, and verify execution placement:
1.	Self-Contained Setup:
o	Register an offline virtual agent pt2-offline-agent with label "isolated-zone".
o	Create a Freestyle job pt2-failover-job bound to "isolated-zone" that writes RAN_ON=${NODE_NAME} to where.txt and archives where.txt (onlyIfSuccessful = true).
2.	Trigger & Stall Verification:
o	Dispatch pt2-failover-job with delay=0sec, capture its Location: queue URL, and wait 2 seconds.
o	Query ${QUEUE_URL}api/json?tree=id,why and assert that the queue item is stalled due to "offline".
3.	Live Label Failover:
o	Rebind pt2-failover-job via Groovy /scriptText to label "built-in" and save the job while the build is still waiting in the queue.
4.	Drain & Placement Proof:
o	Poll until the queued item resolves to a concrete build number and finishes (building == false, result == "SUCCESS").
o	Download where.txt via curl -f and assert that it contains RAN_ON=built-in.
o	Clean up pt2-offline-agent via Groovy at the end of the test.



Challenge 1: The Self-Healing Fleet Capacity & Admission Controller

Scenario
Your platform runs high-priority payment batch jobs (payment-batch-worker). During peak hours, upstream microservices frequently flood Jenkins with requests. If 2 or more builds are already sitting in the queue, new batch submissions must be rejected immediately at the admission gate (429-style backpressure) rather than piling up and timing out.
Your Mission
Write a standalone, self-contained Bash script (lesson11_challenge1_admission.sh) that:
1.	Bootstraps from Scratch: Creates a Freestyle job payment-batch-worker via /scriptText that accepts BATCH_ID, sleeps 4 seconds (to simulate real processing time), writes BATCH_ID=${BATCH_ID} to batch.receipt, and archives batch.receipt (onlyIfSuccessful = true).
2.	Simulates Traffic Burst: Fires 3 builds back-to-back (BATCH-1, BATCH-2, BATCH-3) with delay=0sec so 1 build occupies the executor and 2 builds wait in /queue/api/json.
3.	Enforces Admission Control on Build #4: Before dispatching BATCH-4, queries /queue/api/json?tree=items[id,task[name]] and counts active queued items:
o	If QUEUE_COUNT >= 2, your controller must refuse dispatch, print [ADMISSION DENIED] Queue saturation threshold reached (${QUEUE_COUNT} items waiting), and exit cleanly without queuing BATCH-4.
4.	Drains & Verifies: Waits for BATCH-1 through BATCH-3 to finish and verifies via /job/payment-batch-worker/api/json?tree=builds[number] that total build count did not exceed 3.



Challenge 2: The Automated Degraded-Node Quarantine & Rebalance Pipeline

Scenario
You operate a two-node pool: pool-node-1 (labeled worker-pool) and the built-in controller node (also labeled worker-pool as standby). During a deployment run, pool-node-1 begins failing health checks. Your orchestrator must detect the degraded node, cordon it with an incident reason, verify that subsequent builds automatically route exclusively to the healthy remaining node in worker-pool, and archive proof of node placement.
Your Mission
Write a standalone, self-contained Bash script (lesson11_challenge2_quarantine.sh) that:
1.	Bootstraps Fleet & Job from Scratch:
o	Registers a virtual agent pool-node-1 via /scriptText (numExecutors=1, Mode.NORMAL, labelString="worker-pool").
o	Ensures the built-in node also carries the label "worker-pool" via Groovy (j.setLabelString("worker-pool")).
o	Creates a job pool-health-worker bound to label "worker-pool" that writes EXECUTED_NODE=${NODE_NAME} to placement.txt and archives placement.txt (onlyIfSuccessful = true).
2.	Quarantines Degraded Node (pool-node-1):
o	Inspects /computer/pool-node-1/api/json?tree=temporarilyOffline.
o	If temporarilyOffline is false, sends POST /computer/pool-node-1/toggleOffline?offlineMessage=Degraded%20Node%20Quarantined and verifies "temporarilyOffline":true and "offlineCauseReason" via REST.
3.	Executes & Verifies Rebalanced Routing:
o	Triggers pool-health-worker with delay=0sec, pins its exact build number via the Location: queue header, downloads placement.txt, and asserts that EXECUTED_NODE=built-in (proving Jenkins respected the quarantine and routed around pool-node-1).
4.	Cleanup Hygiene: Uncordons and removes pool-node-1 at the end of the test so your lab environment is left in a clean state.






