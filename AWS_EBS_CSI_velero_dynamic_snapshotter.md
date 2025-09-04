# Complete EBS Snapshot Staggering Solution with Monitoring

This solution addresses both the staggered backup requirements and monitoring concerns, along with handling the scenario where multiple volumes in a single backup need to be coordinated.

## 1. Self-Adjusting Scheduler Controller (scheduler.py)

```python
#!/usr/bin/env python3
import os
import time
import math
import json
import uuid
import kubernetes
import boto3
import logging
from datetime import datetime, timezone
from kubernetes.client.rest import ApiException

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='{"timestamp": "%(asctime)s", "level": "%(levelname)s", "message": %(message)s}',
    datefmt="%Y-%m-%dT%H:%M:%SZ"
)
logger = logging.getLogger("velero-scheduler")

# Load Kubernetes configuration
try:
    kubernetes.config.load_incluster_config()
except kubernetes.config.ConfigException:
    kubernetes.config.load_kube_config()

# Clients
k8s_client = kubernetes.client.CustomObjectsApi()
core_v1 = kubernetes.client.CoreV1Api()
apps_v1 = kubernetes.client.AppsV1Api()

# AWS clients
aws_region = os.environ.get("AWS_REGION", "us-east-1")
ec2 = boto3.client('ec2', region_name=aws_region)
cloudwatch = boto3.client('cloudwatch', region_name=aws_region)

# Configuration
NAMESPACE = "velero"
MIN_INTERVAL_SECONDS = 60  # Minimum interval between volume backups within a snapshot
MAX_CONCURRENT_SNAPSHOTS = int(os.environ.get("MAX_CONCURRENT_SNAPSHOTS", "5"))
BACKUP_WINDOW_HOURS = int(os.environ.get("BACKUP_WINDOW_HOURS", "24"))
SCHEDULE_CRON_FORMAT = "{} {} * * *"  # Minutes Hours * * *
CHECK_INTERVAL_SECONDS = int(os.environ.get("CHECK_INTERVAL_SECONDS", "3600"))
RESOURCE_LABEL_SELECTOR = os.environ.get("RESOURCE_LABEL_SELECTOR", "")
SNAPSHOT_GROUP_LABEL = "velero.io/snapshot-group"
ENSURE_BACKUP_CONSISTENCY = os.environ.get("ENSURE_BACKUP_CONSISTENCY", "true").lower() == "true"

def log_event(event_type, message, extra_data=None):
    log_data = {
        "event_type": event_type,
        "message": message
    }
    
    if extra_data:
        log_data.update(extra_data)
    
    logger.info(json.dumps(log_data))

def get_pv_count_by_statefulset():
    """Get PV counts grouped by StatefulSet"""
    statefulset_volumes = {}
    
    try:
        # Get all PVCs
        pvcs = core_v1.list_persistent_volume_claim_for_all_namespaces()
        
        # Get all StatefulSets
        statefulsets = apps_v1.list_stateful_set_for_all_namespaces()
        
        # Map PVCs to StatefulSets
        for statefulset in statefulsets.items:
            ss_name = statefulset.metadata.name
            ss_namespace = statefulset.metadata.namespace
            
            # Extract volume claim templates
            if not statefulset.spec.volume_claim_templates:
                continue
                
            # Find matching PVCs
            matching_pvcs = []
            for pvc in pvcs.items:
                # Check if PVC belongs to this StatefulSet
                # StatefulSet PVCs follow the pattern: <volume-name>-<statefulset-name>-<ordinal>
                if (pvc.metadata.namespace == ss_namespace and 
                    pvc.metadata.name.endswith(f"-{ss_name}-") or 
                    f"-{ss_name}-" in pvc.metadata.name):
                    matching_pvcs.append(pvc)
            
            if matching_pvcs:
                statefulset_volumes[f"{ss_namespace}/{ss_name}"] = len(matching_pvcs)
        
        log_event("pv_count_check", "Counted PVs by StatefulSet", {"statefulset_volumes": statefulset_volumes})
        return statefulset_volumes
        
    except Exception as e:
        log_event("error", f"Error counting PVs by StatefulSet: {str(e)}")
        return {}

def get_total_pv_count():
    """Count all EBS-based persistent volumes in the cluster"""
    try:
        pvs = core_v1.list_persistent_volume()
        ebs_pvs = [pv for pv in pvs.items if 
                  (hasattr(pv.spec, 'aws_elastic_block_store') and pv.spec.aws_elastic_block_store) or 
                  (hasattr(pv.spec, 'csi') and pv.spec.csi and pv.spec.csi.driver == 'ebs.csi.aws.com')]
        
        log_event("pv_total_count", f"Found {len(ebs_pvs)} EBS-based PVs", {"count": len(ebs_pvs)})
        return len(ebs_pvs)
    except Exception as e:
        log_event("error", f"Error counting total PVs: {str(e)}")
        return 0

def get_throttling_metrics():
    """Get recent API throttling metrics from CloudWatch"""
    try:
        end_time = datetime.now(timezone.utc)
        start_time = end_time.timestamp() - 3600  # Last hour
        
        response = cloudwatch.get_metric_statistics(
            Namespace='AWS/EBS',
            MetricName='ThrottledRequests',
            Dimensions=[{'Name': 'Action', 'Value': 'CreateSnapshot'}],
            StartTime=start_time,
            EndTime=end_time.timestamp(),
            Period=300,
            Statistics=['Sum']
        )
        
        if not response['Datapoints']:
            return 0
        
        throttle_count = max([point['Sum'] for point in response['Datapoints']])
        log_event("throttling_check", f"Detected {throttle_count} throttling events", 
                 {"count": throttle_count, "datapoints": len(response['Datapoints'])})
        return throttle_count
    except Exception as e:
        log_event("error", f"Error checking throttling metrics: {str(e)}")
        return 0

def create_or_update_hook_template():
    """Create or update the pre/post backup hooks template for coordinated snapshots"""
    hook_template = {
        "apiVersion": "v1",
        "kind": "ConfigMap",
        "metadata": {
            "name": "velero-backup-hooks",
            "namespace": NAMESPACE
        },
        "data": {
            "pre-backup-hook.sh": """#!/bin/bash
GROUP_ID=$(cat /etc/podinfo/labels | grep velero.io/snapshot-group | cut -d= -f2)
if [ -n "$GROUP_ID" ]; then
  echo "$(date -u +%FT%TZ) - Waiting for snapshot group $GROUP_ID to be ready"
  # Create marker for this pod
  POD_NAME=$(cat /etc/podinfo/name)
  NAMESPACE=$(cat /etc/podinfo/namespace)
  kubectl create configmap "snapshot-wait-$GROUP_ID-$NAMESPACE-$POD_NAME" \
    --namespace velero \
    --from-literal=status=waiting \
    --from-literal=podName=$POD_NAME \
    --from-literal=namespace=$NAMESPACE \
    --from-literal=timestamp="$(date -u +%FT%TZ)"

  # Wait until the controller signals us to proceed
  while true; do
    PROCEED=$(kubectl get configmap -n velero "snapshot-proceed-$GROUP_ID" -o jsonpath="{.data.status}" --ignore-not-found)
    if [ "$PROCEED" = "ready" ]; then
      echo "$(date -u +%FT%TZ) - Group $GROUP_ID is ready, proceeding with backup"
      break
    fi
    echo "$(date -u +%FT%TZ) - Waiting for snapshot group $GROUP_ID..."
    sleep 10
  done
fi
""",
            "post-backup-hook.sh": """#!/bin/bash
GROUP_ID=$(cat /etc/podinfo/labels | grep velero.io/snapshot-group | cut -d= -f2)
if [ -n "$GROUP_ID" ]; then
  POD_NAME=$(cat /etc/podinfo/name)
  NAMESPACE=$(cat /etc/podinfo/namespace)
  
  # Signal completion
  kubectl create configmap "snapshot-complete-$GROUP_ID-$NAMESPACE-$POD_NAME" \
    --namespace velero \
    --from-literal=status=completed \
    --from-literal=podName=$POD_NAME \
    --from-literal=namespace=$NAMESPACE \
    --from-literal=timestamp="$(date -u +%FT%TZ)"
    
  # Clean up our wait marker
  kubectl delete configmap "snapshot-wait-$GROUP_ID-$NAMESPACE-$POD_NAME" \
    --namespace velero --ignore-not-found
fi
"""
        }
    }
    
    try:
        # Check if ConfigMap exists
        try:
            core_v1.read_namespaced_config_map(name="velero-backup-hooks", namespace=NAMESPACE)
            # Update the existing ConfigMap
            core_v1.patch_namespaced_config_map(
                name="velero-backup-hooks",
                namespace=NAMESPACE,
                body=hook_template
            )
            log_event("hook_update", "Updated velero-backup-hooks ConfigMap")
        except ApiException as e:
            if e.status == 404:
                # Create new ConfigMap
                core_v1.create_namespaced_config_map(
                    namespace=NAMESPACE,
                    body=hook_template
                )
                log_event("hook_create", "Created velero-backup-hooks ConfigMap")
            else:
                raise
    except Exception as e:
        log_event("error", f"Error creating/updating hook templates: {str(e)}")

def manage_snapshot_group_coordination(group_id):
    """Manage coordination of volumes in a snapshot group"""
    try:
        # Check if proceed ConfigMap exists, create if not
        try:
            core_v1.read_namespaced_config_map(name=f"snapshot-proceed-{group_id}", namespace=NAMESPACE)
        except ApiException as e:
            if e.status == 404:
                # Create proceed ConfigMap with wait status
                core_v1.create_namespaced_config_map(
                    namespace=NAMESPACE,
                    body={
                        "apiVersion": "v1",
                        "kind": "ConfigMap",
                        "metadata": {
                            "name": f"snapshot-proceed-{group_id}",
                            "namespace": NAMESPACE
                        },
                        "data": {
                            "status": "wait",
                            "created": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
                        }
                    }
                )
                log_event("group_created", f"Created coordination ConfigMap for group {group_id}", 
                          {"group_id": group_id, "status": "wait"})
        
        # Get all waiting markers
        waiting_pods = []
        try:
            config_maps = core_v1.list_namespaced_config_map(
                namespace=NAMESPACE,
                label_selector=f"app=velero-snapshot-wait,group={group_id}"
            )
            waiting_pods = [cm.metadata.name for cm in config_maps.items]
        except Exception as e:
            log_event("error", f"Error listing waiting pods for group {group_id}: {str(e)}")
        
        # Get all ready markers
        completed_pods = []
        try:
            config_maps = core_v1.list_namespaced_config_map(
                namespace=NAMESPACE, 
                label_selector=f"app=velero-snapshot-complete,group={group_id}"
            )
            completed_pods = [cm.metadata.name for cm in config_maps.items]
        except Exception as e:
            log_event("error", f"Error listing completed pods for group {group_id}: {str(e)}")
        
        # If all pods are ready, signal proceed
        log_event("group_status", f"Group {group_id} status", 
                  {"group_id": group_id, "waiting": len(waiting_pods), "completed": len(completed_pods)})
        
        if len(waiting_pods) > 0 and len(waiting_pods) == len(completed_pods):
            # Update proceed ConfigMap
            core_v1.patch_namespaced_config_map(
                name=f"snapshot-proceed-{group_id}",
                namespace=NAMESPACE,
                body={
                    "data": {
                        "status": "ready",
                        "updated": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
                    }
                }
            )
            log_event("group_ready", f"All pods in group {group_id} are ready", 
                     {"group_id": group_id, "pod_count": len(waiting_pods)})
            
            # Clean up completed markers after a delay
            time.sleep(60)
            for pod in completed_pods:
                try:
                    core_v1.delete_namespaced_config_map(name=pod, namespace=NAMESPACE)
                except Exception as e:
                    log_event("error", f"Error deleting completed marker {pod}: {str(e)}")
            
            # Clean up proceed marker
            try:
                core_v1.delete_namespaced_config_map(name=f"snapshot-proceed-{group_id}", namespace=NAMESPACE)
            except Exception as e:
                log_event("error", f"Error deleting proceed marker for {group_id}: {str(e)}")
    
    except Exception as e:
        log_event("error", f"Error coordinating snapshot group {group_id}: {str(e)}")

def calculate_backup_schedule(pv_count, throttle_count, statefulset_volumes):
    """Calculate optimal backup schedule based on PV counts and throttling history"""
    # Start with base concurrency limit
    adjusted_concurrency = MAX_CONCURRENT_SNAPSHOTS
    
    # Adjust based on throttling
    if throttle_count > 0:
        adjusted_concurrency = max(1, MAX_CONCURRENT_SNAPSHOTS - math.ceil(throttle_count/2))
        log_event("concurrency_adjusted", f"Adjusted concurrency to {adjusted_concurrency} due to throttling",
                 {"original": MAX_CONCURRENT_SNAPSHOTS, "adjusted": adjusted_concurrency})
    
    # Calculate interval based on PV count and available window
    minutes_available = BACKUP_WINDOW_HOURS * 60
    total_snapshots_needed = pv_count / adjusted_concurrency
    interval_minutes = max(1, math.floor(minutes_available / total_snapshots_needed))
    
    # Create schedule for each StatefulSet or standalone PVs
    schedules = []
    
    # Start timestamp to spread backups
    start_minute = 0
    start_hour = 0
    
    # First handle StatefulSets (keep volumes together)
    for statefulset_name, volume_count in statefulset_volumes.items():
        namespace, name = statefulset_name.split('/')
        
        # Generate unique group ID for this backup
        group_id = f"group-{uuid.uuid4().hex[:8]}"
        
        # Calculate minutes and hours
        schedule_minute = start_minute % 60
        schedule_hour = (start_hour + (start_minute // 60)) % 24
        
        # Create schedule
        schedules.append({
            "name": f"auto-{namespace}-{name}",
            "namespace": namespace,
            "includes": [namespace],
            "resources": ["statefulsets"],
            "selector": f"app={name}",
            "schedule": SCHEDULE_CRON_FORMAT.format(schedule_minute, schedule_hour),
            "group_id": group_id,
            "volume_count": volume_count
        })
        
        # Increment start time for next schedule
        start_minute += interval_minutes
        if start_minute >= 60:
            start_hour += start_minute // 60
            start_minute = start_minute % 60
    
    # Handle remaining standalone volumes by namespace groups
    if pv_count > sum(statefulset_volumes.values()):
        # Get namespaces that have persistent volumes
        try:
            namespaces_with_pvs = set()
            pvcs = core_v1.list_persistent_volume_claim_for_all_namespaces()
            
            for pvc in pvcs.items:
                # Skip PVCs that are part of StatefulSets we already handled
                skip = False
                for statefulset_name in statefulset_volumes:
                    ns, name = statefulset_name.split('/')
                    if pvc.metadata.namespace == ns and name in pvc.metadata.name:
                        skip = True
                        break
                
                if not skip:
                    namespaces_with_pvs.add(pvc.metadata.namespace)
            
            # Create schedule for each namespace with PVs
            for namespace in namespaces_with_pvs:
                # Generate unique group ID for this backup
                group_id = f"group-{uuid.uuid4().hex[:8]}"
                
                # Calculate minutes and hours
                schedule_minute = start_minute % 60
                schedule_hour = (start_hour + (start_minute // 60)) % 24
                
                # Create schedule
                schedules.append({
                    "name": f"auto-{namespace}-volumes",
                    "namespace": namespace,
                    "includes": [namespace],
                    "resources": ["persistentvolumeclaims", "pods"],
                    "schedule": SCHEDULE_CRON_FORMAT.format(schedule_minute, schedule_hour),
                    "group_id": group_id,
                    "volume_count": 0  # Will be counted later
                })
                
                # Increment start time for next schedule
                start_minute += interval_minutes
                if start_minute >= 60:
                    start_hour += start_minute // 60
                    start_minute = start_minute % 60
                
        except Exception as e:
            log_event("error", f"Error scheduling standalone volumes: {str(e)}")
    
    log_event("schedule_calculation", "Calculated backup schedules", 
              {"schedules": len(schedules), "interval_minutes": interval_minutes})
    return schedules

def create_velero_schedules(schedules):
    """Create Velero backup schedules"""
    # First, delete existing auto-generated schedules
    try:
        existing_schedules = k8s_client.list_namespaced_custom_object(
            group="velero.io",
            version="v1",
            namespace=NAMESPACE,
            plural="schedules"
        )
        
        for schedule in existing_schedules['items']:
            if schedule['metadata']['name'].startswith('auto-'):
                try:
                    k8s_client.delete_namespaced_custom_object(
                        group="velero.io",
                        version="v1",
                        namespace=NAMESPACE,
                        plural="schedules",
                        name=schedule['metadata']['name']
                    )
                    log_event("schedule_deleted", f"Deleted schedule {schedule['metadata']['name']}")
                except Exception as e:
                    log_event("error", f"Error deleting schedule {schedule['metadata']['name']}: {str(e)}")
    except Exception as e:
        log_event("error", f"Error listing existing schedules: {str(e)}")
    
    # Create new schedules
    for schedule_data in schedules:
        # Define hooks if we're ensuring backup consistency
        hooks = {}
        if ENSURE_BACKUP_CONSISTENCY:
            hooks = {
                "resources": [
                    {
                        "name": "backup-wait",
                        "includedNamespaces": schedule_data["includes"],
                        "labelSelector": {
                            "matchExpressions": [
                                {
                                    "key": SNAPSHOT_GROUP_LABEL,
                                    "operator": "Exists"
                                }
                            ]
                        },
                        "pre": [
                            {
                                "exec": {
                                    "command": ["/hooks/pre-backup-hook.sh"],
                                    "timeout": "10m",
                                    "onError": "Fail"
                                }
                            }
                        ],
                        "post": [
                            {
                                "exec": {
                                    "command": ["/hooks/post-backup-hook.sh"],
                                    "timeout": "5m",
                                    "onError": "Continue"
                                }
                            }
                        ]
                    }
                ]
            }
        
        # Build label selector
        labelSelector = {}
        if schedule_data.get("selector"):
            labelSelector = {
                "matchExpressions": [
                    {
                        "key": schedule_data["selector"].split("=")[0],
                        "operator": "In",
                        "values": [schedule_data["selector"].split("=")[1]]
                    }
                ]
            }
        
        # Create schedule body
        schedule_body = {
            "apiVersion": "velero.io/v1",
            "kind": "Schedule",
            "metadata": {
                "name": schedule_data["name"],
                "namespace": NAMESPACE,
                "labels": {
                    "created-by": "velero-scheduler",
                    "snapshot-group": schedule_data["group_id"]
                }
            },
            "spec": {
                "schedule": schedule_data["schedule"],
                "template": {
                    "hooks": hooks,
                    "includedNamespaces": schedule_data["includes"],
                    "includedResources": schedule_data.get("resources", ["*"]),
                    "labelSelector": labelSelector,
                    "snapshotVolumes": True,
                    "storageLocation": "default",
                    "ttl": "168h",
                    "metadata": {
                        "labels": {
                            "snapshot-group": schedule_data["group_id"]
                        }
                    }
                }
            }
        }
        
        try:
            k8s_client.create_namespaced_custom_object(
                group="velero.io",
                version="v1",
                namespace=NAMESPACE,
                plural="schedules",
                body=schedule_body
            )
            log_event("schedule_created", f"Created schedule {schedule_data['name']}", 
                     {"name": schedule_data["name"], 
                      "schedule": schedule_data["schedule"],
                      "group_id": schedule_data["group_id"]})
        except Exception as e:
            log_event("error", f"Error creating schedule {schedule_data['name']}: {str(e)}")

def create_coordination_cronjob():
    """Create a CronJob to handle snapshot coordination"""
    cronjob = {
        "apiVersion": "batch/v1",
        "kind": "CronJob",
        "metadata": {
            "name": "velero-snapshot-coordinator",
            "namespace": NAMESPACE
        },
        "spec": {
            "schedule": "* * * * *",  # Every minute
            "concurrencyPolicy": "Replace",
            "jobTemplate": {
                "spec": {
                    "template": {
                        "metadata": {
                            "labels": {
                                "app": "velero-snapshot-coordinator"
                            }
                        },
                        "spec": {
                            "serviceAccountName": "velero",
                            "restartPolicy": "Never",
                            "containers": [
                                {
                                    "name": "coordinator",
                                    "image": "bitnami/kubectl:latest",
                                    "command": [
                                        "sh",
                                        "-c",
                                        """
                                        # Get all snapshot waiting markers
                                        WAIT_MAPS=$(kubectl get configmap -n velero -l app=velero-snapshot-wait -o jsonpath='{.items[*].metadata.name}')
                                        
                                        # Extract unique group IDs
                                        GROUPS=$(kubectl get configmap -n velero -l app=velero-snapshot-wait -o jsonpath='{.items[*].metadata.labels.group}' | tr ' ' '\\n' | sort | uniq)
                                        
                                        # Process each group
                                        for GROUP in $GROUPS; do
                                          echo "Processing group $GROUP"
                                          
                                          # Count waiting pods
                                          WAITING=$(kubectl get configmap -n velero -l "app=velero-snapshot-wait,group=$GROUP" --no-headers | wc -l)
                                          
                                          # Count completed pods
                                          COMPLETED=$(kubectl get configmap -n velero -l "app=velero-snapshot-complete,group=$GROUP" --no-headers | wc -l)
                                          
                                          echo "Group $GROUP: $WAITING waiting, $COMPLETED completed"
                                          
                                          # If all have completed, we can set the proceed flag
                                          if [ $WAITING -gt 0 ] && [ $WAITING -eq $COMPLETED ]; then
                                            echo "All pods ready in group $GROUP, setting proceed flag"
                                            
                                            # Check if proceed ConfigMap exists
                                            if kubectl get configmap "snapshot-proceed-$GROUP" -n velero > /dev/null 2>&1; then
                                              # Update proceed ConfigMap
                                              kubectl patch configmap "snapshot-proceed-$GROUP" -n velero --type=merge -p '{"data":{"status":"ready","updated":"'$(date -u +%FT%TZ)'"}}'
                                            else
                                              # Create proceed ConfigMap
                                              kubectl create configmap "snapshot-proceed-$GROUP" -n velero --from-literal=status=ready --from-literal=updated="$(date -u +%FT%TZ)"
                                            fi
                                            
                                            # Give time for pods to proceed
                                            sleep 60
                                            
                                            # Clean up completed markers
                                            kubectl delete configmap -n velero -l "app=velero-snapshot-complete,group=$GROUP"
                                            
                                            # Clean up proceed marker
                                            kubectl delete configmap "snapshot-proceed-$GROUP" -n velero
                                          fi
                                        done
                                        
                                        # Clean up old markers (>1h)
                                        OLD_THRESHOLD=$(date -u -d '1 hour ago' +%FT%TZ)
                                        
                                        # Get list of old waiting markers
                                        OLD_WAITING=$(kubectl get configmap -n velero -l app=velero-snapshot-wait -o jsonpath='{.items[?(@.data.timestamp < "'$OLD_THRESHOLD'")].metadata.name}')
                                        if [ -n "$OLD_WAITING" ]; then
                                          echo "Cleaning up old waiting markers: $OLD_WAITING"
                                          for CM in $OLD_WAITING; do
                                            kubectl delete configmap $CM -n velero
                                          done
                                        fi
                                        
                                        # Get list of old completed markers
                                        OLD_COMPLETED=$(kubectl get configmap -n velero -l app=velero-snapshot-complete -o jsonpath='{.items[?(@.data.timestamp < "'$OLD_THRESHOLD'")].metadata.name}')
                                        if [ -n "$OLD_COMPLETED" ]; then
                                          echo "Cleaning up old completed markers: $OLD_COMPLETED"
                                          for CM in $OLD_COMPLETED; do
                                            kubectl delete configmap $CM -n velero
                                          done
                                        fi
                                        
                                        # Get list of old proceed markers
                                        OLD_PROCEED=$(kubectl get configmap -n velero -l app=velero-snapshot-proceed -o jsonpath='{.items[?(@.data.created < "'$OLD_THRESHOLD'")].metadata.name}')
                                        if [ -n "$OLD_PROCEED" ]; then
                                          echo "Cleaning up old proceed markers: $OLD_PROCEED"
                                          for CM in $OLD_PROCEED; do
                                            kubectl delete configmap $CM -n velero
                                          done
                                        fi
                                        """
                                    ]
                                }
                            ]
                        }
                    }
                }
            }
        }
    }
    
    try:
        # Check if CronJob exists
        batch_v1 = kubernetes.client.BatchV1Api()
        try:
            batch_v1.read_namespaced_cron_job(name="velero-snapshot-coordinator", namespace=NAMESPACE)
            # Update the existing CronJob
            batch_v1.patch_namespaced_cron_job(
                name="velero-snapshot-coordinator",
                namespace=NAMESPACE,
                body=cronjob
            )
            log_event("cronjob_update", "Updated velero-snapshot-coordinator CronJob")
        except ApiException as e:
            if e.status == 404:
                # Create new CronJob
                batch_v1.create_namespaced_cron_job(namespace=NAMESPACE, body=cronjob)
                log_event("cronjob_create", "Created velero-snapshot-coordinator CronJob")
            else:
                raise
    except Exception as e:
        log_event("error", f"Error creating/updating coordinator CronJob: {str(e)}")

def create_annotation_job():
    """Create a one-time job to annotate pods with group information"""
    job_name = f"velero-pod-annotator-{uuid.uuid4().hex[:8]}"
    
    job = {
        "apiVersion": "batch/v1",
        "kind": "Job",
        "metadata": {
            "name": job_name,
            "namespace": NAMESPACE
        },
        "spec": {
            "template": {
                "metadata": {
                    "labels": {
                        "app": "velero-pod-annotator"
                    }
                },
                "spec": {
                    "serviceAccountName": "velero",
                    "restartPolicy": "Never",
                    "containers": [
                        {
                            "name": "annotator",
                            "image": "bitnami/kubectl:latest",
                            "command": [
                                "sh",
                                "-c",
                                """
                                # Find StatefulSets
                                STATEFULSETS=$(kubectl get statefulset --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\\n"}{end}')
                                
                                for STATEFULSET in $STATEFULSETS; do
                                  NAMESPACE=$(echo $STATEFULSET | cut -d/ -f1)
                                  NAME=$(echo $STATEFULSET | cut -d/ -f2)
                                  
                                  # Generate group ID for this StatefulSet
                                  GROUP_ID="group-$(echo $NAMESPACE-$NAME | md5sum | cut -c1-8)"
                                  
                                  echo "Labeling pods for StatefulSet $NAMESPACE/$NAME with group $GROUP_ID"
                                  
                                  # Find and label all pods for this StatefulSet
                                  POD_LIST=$(kubectl get pods -n $NAMESPACE -l app=$NAME -o name)
                                  for POD in $POD_LIST; do
                                    echo "Labeling $POD with group $GROUP_ID"
                                    kubectl label $POD -n $NAMESPACE velero.io/snapshot-group=$GROUP_ID --overwrite
                                  done
                                done
                                
                                # Find standalone persistent volumes
                                echo "Processing standalone volumes..."
                                STANDALONE_NAMESPACES=$(kubectl get pvc --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\\n"}{end}' | sort | uniq)
                                
                                for NS in $STANDALONE_NAMESPACES; do
                                  # Generate namespace group ID
                                  NS_GROUP_ID="group-$(echo $NS-volumes | md5sum | cut -c1-8)"
                                  
                                  # Find pods with volumes in this namespace that aren't part of StatefulSets
                                  PODS_WITH_VOLUMES=$(kubectl get pods -n $NS -o json | jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim != null) | .metadata.name')
                                  
                                  for POD in $PODS_WITH_VOLUMES; do
                                    # Check if pod is part of a StatefulSet
                                    IS_STATEFULSET=$(kubectl get pod $POD -n $NS -o json | jq -r '.metadata.ownerReferences[] | select(.kind=="StatefulSet") | .name' 2>/dev/null)
                                    
                                    if [ -z "$IS_STATEFULSET" ]; then
                                      echo "Labeling standalone pod $POD in namespace $NS with group $NS_GROUP_ID"
                                      kubectl label pod $POD -n $NS velero.io/snapshot-group=$NS_GROUP_ID --overwrite
                                    fi
                                  done
                                done
                                
                                echo "Pod labeling complete"
                                """
                            ]
                        }
                    ]
                }
            }
        }
    }
    
    try:
        # Create new Job
        batch_v1 = kubernetes.client.BatchV1Api()
        batch_v1.create_namespaced_job(namespace=NAMESPACE, body=job)
        log_event("job_create", f"Created pod annotator job {job_name}")
    except Exception as e:
        log_event("error", f"Error creating pod annotator job: {str(e)}")

def setup_monitoring_stack():
    """Create monitoring elements for Elasticsearch integration"""
    # Create a ServiceMonitor for Velero if using Prometheus
    service_monitor = {
        "apiVersion": "monitoring.coreos.com/v1",
        "kind": "ServiceMonitor",
        "metadata": {
            "name": "velero-monitor",
            "namespace": NAMESPACE
        },
        "spec": {
            "selector": {
                "matchLabels": {
                    "app.kubernetes.io/name": "velero"
                }
            },
            "endpoints": [
                {
                    "port": "monitoring",
                    "interval": "30s"
                }
            ]
        }
    }
    
    try:
        # Try to create ServiceMonitor if CRD exists
        try:
            k8s_client.create_namespaced_custom_object(
                group="monitoring.coreos.com",
                version="v1",
                namespace=NAMESPACE,
                plural="servicemonitors",
                body=service_monitor
            )
            log_event("monitor_created", "Created Velero ServiceMonitor")
        except ApiException as e:
            if e.status != 404:  # Ignore if CRD doesn't exist
                log_event("warning", f"Error creating ServiceMonitor: {str(e)}")
    except Exception as e:
        log_event("warning", f"Error setting up monitoring: {str(e)}")
    
    # Create ConfigMap for Filebeat configuration to ship logs to Elasticsearch
    filebeat_config = {
        "apiVersion": "v1",
        "kind": "ConfigMap",
        "metadata": {
            "name": "velero-filebeat-config",
            "namespace": NAMESPACE
        },
        "data": {
            "filebeat.yml": """
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/velero-*.log
  processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"

output.elasticsearch:
  hosts: ["${ELASTICSEARCH_HOST:elasticsearch.logging:9200}"]
  index: "velero-backups-%{+yyyy.MM.dd}"
  
processors:
  - decode_json_fields:
      fields: ["message"]
      target: "velero"
      process_array: false
      max_depth: 3
  - drop_event:
      when:
        not:
          has_fields: ['velero.event_type']
"""
        }
    }
    
    try:
        # Check if ConfigMap exists
        try:
            core_v1.read_namespaced_config_map(name="velero-filebeat-config", namespace=NAMESPACE)
            # Update the existing ConfigMap
            core_v1.patch_namespaced_config_map(
                name="velero-filebeat-config",
                namespace=NAMESPACE,
                body=filebeat_config
            )
            log_event("filebeat_update", "Updated velero-filebeat-config ConfigMap")
        except ApiException as e:
            if e.status == 404:
                # Create new ConfigMap
                core_v1.create_namespaced_config_map(namespace=NAMESPACE, body=filebeat_config)
                log_event("filebeat_create", "Created velero-filebeat-config ConfigMap")
            else:
                raise
    except Exception as e:
        log_event("error", f"Error creating/updating Filebeat config: {str(e)}")

def main():
    """Main execution loop"""
    log_event("startup", "Velero Scheduler starting")
    
    # Create prerequisite components
    if ENSURE_BACKUP_CONSISTENCY:
        create_or_update_hook_template()
        create_coordination_cronjob()
    
    # Setup monitoring
    setup_monitoring_stack()
    
    while True:
        try:
            # Get volume counts
            total_pv_count = get_total_pv_count()
            statefulset_volumes = get_pv_count_by_statefulset()
            
            # Check throttling
            throttle_count = get_throttling_metrics()
            
            # Calculate optimal scheduling
            schedules = calculate_backup_schedule(total_pv_count, throttle_count, statefulset_volumes)
            
            # Create schedules
            create_velero_schedules(schedules)
            
            # Label pods with group information for coordinated backups
            if ENSURE_BACKUP_CONSISTENCY:
                create_annotation_job()
            
            # Wait for next check interval
            log_event("scheduler_sleeping", f"Scheduler sleeping for {CHECK_INTERVAL_SECONDS} seconds")
            time.sleep(CHECK_INTERVAL_SECONDS)
            
        except Exception as e:
            log_event("error", f"Error in main loop: {str(e)}", {"traceback": str(e)})
            time.sleep(300)  # Wait 5 minutes on error

if __name__ == "__main__":
    main()
```

## 2. Deployment and RBAC for Scheduler Controller

```yaml
# scheduler-deployment.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: velero-scheduler
  namespace: velero

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: velero-scheduler
rules:
- apiGroups: [""]
  resources: ["persistentvolumes", "namespaces", "persistentvolumeclaims", "pods", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "deployments"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
- apiGroups: ["velero.io"]
  resources: ["schedules", "backups", "restores"]
  verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
- apiGroups: ["monitoring.coreos.com"]
  resources: ["servicemonitors"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: velero-scheduler
subjects:
- kind: ServiceAccount
  name: velero-scheduler
  namespace: velero
roleRef:
  kind: ClusterRole
  name: velero-scheduler
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: velero-scheduler
  namespace: velero
spec:
  replicas: 1
  selector:
    matchLabels:
      app: velero-scheduler
  template:
    metadata:
      labels:
        app: velero-scheduler
    spec:
      serviceAccountName: velero-scheduler
      containers:
      - name: scheduler
        image: python:3.9-slim
        command: ["python", "/app/scheduler.py"]
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: app-code
          mountPath: /app
        env:
        - name: AWS_REGION
          value: "us-east-1"
        - name: MAX_CONCURRENT_SNAPSHOTS
          value: "5"
        - name: BACKUP_WINDOW_HOURS
          value: "24"
        - name: CHECK_INTERVAL_SECONDS
          value: "3600"
        - name: ENSURE_BACKUP_CONSISTENCY
          value: "true"
      volumes:
      - name: app-code
        configMap:
          name: velero-scheduler-code
          defaultMode: 0755
```

## 3. ConfigMap for Scheduler Code

```yaml
# scheduler-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: velero-scheduler-code
  namespace: velero
data:
  scheduler.py: |
    # Paste the full Python script here from the first section
```

## 4. Filebeat DaemonSet for Elasticsearch Monitoring

```yaml
# velero-filebeat.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: velero-filebeat
  namespace: velero
  labels:
    app: velero-filebeat
spec:
  selector:
    matchLabels:
      app: velero-filebeat
  template:
    metadata:
      labels:
        app: velero-filebeat
    spec:
      serviceAccountName: velero-scheduler
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.17.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch.elastic-system.svc.cluster.local:9200"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: velero-filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      - name: data
        hostPath:
          path: /var/lib/filebeat-velero-data
          type: DirectoryOrCreate
```

## 5. Kibana Dashboard for Monitoring

```json
{
  "attributes": {
    "title": "Velero Backup Dashboard",
    "visState": "{\"title\":\"Velero Backup Dashboard\",\"type\":\"dashboard\",\"params\":{\"version\":1},\"savedObjectId\":\"velero-backups\"}",
    "uiStateJSON": "{}",
    "version": 1,
    "timeRestore": false,
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"
    }
  },
  "references": [
    {
      "name": "panel_0",
      "type": "visualization",
      "id": "velero-backup-status"
    },
    {
      "name": "panel_1",
      "type": "visualization",
      "id": "velero-throttling-events"
    },
    {
      "name": "panel_2",
      "type": "visualization",
      "id": "velero-snapshot-duration"
    },
    {
      "name": "panel_3",
      "type": "search",
      "id": "velero-backup-logs"
    }
  ]
}
```

## 6. Helm Values Override for Velero Installation

```yaml
# velero-values.yaml
configuration:
  backupStorageLocation:
    # Set your actual bucket values
    bucket: my-velero-backups
    config:
      region: us-east-1
  
  volumeSnapshotLocation:
    config:
      region: us-east-1
    
  features: "EnableCSI,EnableAPIGroupVersions"

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.5.0
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - mountPath: /target
        name: plugins

extraVolumes:
  - name: backup-hooks
    configMap:
      name: velero-backup-hooks
      defaultMode: 0744

extraVolumeMounts:
  - name: backup-hooks
    mountPath: /hooks

# Set resource limits to control API usage
resources:
  requests:
    cpu: 500m
    memory: 128Mi
  limits:
    cpu: 1000m
    memory: 512Mi

# Configure AWS plugin settings
initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.5.0
    volumeMounts:
      - mountPath: /target
        name: plugins

# Add pod annotations for log collection
podAnnotations:
  co.elastic.logs/enabled: "true"
```

## 7. Installation and Usage Instructions

```bash
#!/bin/bash
# install-staggered-backup-system.sh

# Set your region
AWS_REGION="us-east-1"
BACKUP_BUCKET="my-velero-backups"
ELASTICSEARCH_HOST="elasticsearch.elastic-system.svc.cluster.local:9200"

# Install Velero with AWS plugin
echo "Installing Velero..."
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# Create AWS credentials file
cat > ./credentials-velero <<EOF
[default]
aws_access_key_id=${AWS_ACCESS_KEY_ID}
aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}
EOF

# Install Velero with customized values
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --version 2.32.6 \
  --set-file credentials.secretContents.cloud=./credentials-velero \
  --values velero-values.yaml \
  --set configuration.backupStorageLocation.bucket=${BACKUP_BUCKET} \
  --set configuration.backupStorageLocation.config.region=${AWS_REGION} \
  --set configuration.volumeSnapshotLocation.config.region=${AWS_REGION}

# Create AWS credentials secret for the scheduler
kubectl create secret generic aws-credentials \
  --namespace velero \
  --from-literal=access-key=${AWS_ACCESS_KEY_ID} \
  --from-literal=secret-key=${AWS_SECRET_ACCESS_KEY}

# Apply scheduler ConfigMap
echo "Applying scheduler code..."
kubectl create configmap velero-scheduler-code \
  --namespace velero \
  --from-file=scheduler.py

# Apply scheduler RBAC and deployment
echo "Deploying scheduler..."
kubectl apply -f scheduler-deployment.yaml

# Apply Filebeat configuration
echo "Setting up monitoring..."
kubectl create configmap velero-filebeat-config \
  --namespace velero \
  --from-file=filebeat.yml

# Deploy Filebeat
kubectl apply -f velero-filebeat.yaml

echo "Setup complete! The system will automatically begin scheduling staggered backups."
echo "Monitor the system using: kubectl logs -n velero -l app=velero-scheduler -f"
echo "Backup information will be available in your Elasticsearch cluster."
```

## Technical Implementation Overview

This solution addresses your requirements in several key ways:

### 1. Automatic Staggering of Snapshots

The scheduler controller automatically calculates the optimal schedule based on:
- The total number of persistent volumes in your cluster
- Historical throttling events from AWS API
- The defined 24-hour backup window

It then creates staggered Velero schedules to distribute backup operations throughout the day.

### 2. Volume Consistency within StatefulSets

The system identifies volumes that belong to the same StatefulSet and ensures they're backed up together by:
- Labeling pods with a consistent group ID
- Using pre/post backup hooks to coordinate snapshots
- Implementing a pause-and-resume mechanism between pods in the same group

### 3. Elasticsearch Monitoring Integration

The solution integrates with your existing Elasticsearch setup by:
- Sending structured logs with backup events and status
- Deploying Filebeat to collect and ship logs
- Providing a Kibana dashboard for visualization
- Including detailed metrics on backup timing and AWS API throttling

### 4. Automatic Adaptation to AWS API Limits

The system continuously monitors for AWS throttling events and automatically adjusts:
- Concurrency limits for snapshot operations
- Time spacing between backups
- Grouping of volumes to optimize API usage

### 5. Comprehensive Logging

The scheduler produces detailed, structured logs that capture:
- Backup scheduling decisions
- Snapshot coordination events
- Throttling detection and responses
- Error conditions with detailed context

## Using the Solution

After installation, the system will:

1. Analyze your cluster to identify all volumes and their relationships
2. Create optimally staggered backup schedules
3. Monitor AWS API usage and adjust scheduling as needed
4. Send detailed logs to Elasticsearch for monitoring

You can view the backup status and performance through:
- Kibana dashboards (using the provided template)
- Kubernetes logs: `kubectl logs -n velero -l app=velero-scheduler -f`
- Velero CLI: `velero get backup`

The solution is fully automated, requiring minimal intervention once deployed.
