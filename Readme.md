# Enable stateful workloads on Kubernetes on public clouds with NetApp Cloud Volumes and Trident CSI

### Present CVS and CVO
### Present CSI provisioner Trident 
### Show PVC provisioning with Trident on GKE

## Use cases
### RWX PVC, data sharing. CA workloads, TensorFlow, BA/BI workloads
Demo:
```bash
alias bat='bat -n --paging=never'
# Create 1TB PVC
bat nfs-pvc-shared.yaml
kubectl apply -f nfs-pvc-shared.yaml
# Download SQL demo dataset into PV
bat fetchsampledata.yaml
kubectl apply -f fetchsampledata.yaml
# Create a helper pod which gives access to PV under /data
bat pvc-helper.yaml
kubectl apply -f pvc-helper.yaml
kubectl wait --for=condition=Ready pod/pvc-helper
kubectl exec -t pvc-helper -- ls -lR /data
```
### Resize (expand) a PV
Demo:
```bash
# Dynamically change size of existing PVC to 2Ti
kubectl get pvc nfs-shared 
kubectl patch pvc nfs-shared -p '{"spec":{"resources":{"requests":{"storage":"2Ti"}}}}'
sleep 3
kubectl get pvc nfs-shared
# Size changed. Win !!!
```
### Creating and using Snapshots
Demo:
```bash
# Using snapshots
# Pre-work: Create snapshotclass. Need to be done once per cluster
bat snapshotclass.yaml
kubectl apply -f snapshotclass.yaml
# Create snapshot of PVC nfs-shared
kubectl get pvc nfs-shared
bat nfs-pvc-shared-snap01.yaml
kubectl apply -f nfs-pvc-shared-snap01.yaml
kubectl get VolumeSnapshot
# Have a look into the PV
kubectl exec -ti pvc-helper sh
cd /data
ls -la
# Please not the .snapshot directory
ls .snapshot
ls .snapshot/<snapUUID>
# snapshot are read-only
rm .snapshot/<snapUUID>/test_db/employees.sql
# let's delete a file
rm test_db/employees.sql
ls test_db/employees.sql
# Aaargh, just deleted my employee file.
# Let's recover it
cp .snapshot/<snapUUID>/test_db/employees.sql test_db
ls -l test_db/employees.sql
# Win !!!
```
### Create database on RWO PVC
Demo:
```bash
# Start database on new PVC
bat database-prod.yaml
kubectl apply -f database-prod.yaml
# Import sample data into mariadb database
kubectl exec -ti $(kubectl get pods -l app=mariadb -o jsonpath="{.items[].metadata.name}") -- bash
cd /data/test_db
mysql -h mariadb -ppassword < employees.sql
exit
# run a query
kubectl run -it --rm --image=mariadb:10 --restart=Never mysql-client -- mysql -h mariadb -ppassword employees
select * from employees where first_name="Danel";
select count(*) from employees;
exit
```

### Create a clone of the database
Use cases: instantly create up to date r/w copies of big data sets. Think CI/CD, DEV/QA
A clone is done by creating a new PVC with an Snapshot as "data template".
Demo:
```bash
# Create *consistant* snapshot of database
kubectl get deployments
# Lock table
kubectl run -ti --rm --image=mariadb:10 --restart=Never mysql-client -- mysql -h mariadb -ppassword employees -e 'FLUSH TABLES WITH READ LOCK;'
# Create snapshot
kubectl get VolumeSnapshot
bat database-snapshot.yaml
kubectl apply -f database-snapshot.yaml
# Unlock table
kubectl run -ti --rm --image=mariadb:10 --restart=Never mysql-client -- mysql -h mariadb -ppassword employees -e 'UNLOCK TABLES;'
kubectl get VolumeSnapshot
# Create TEST deployment using clone
bat database-test.yaml
kubectl apply -f database-test.yaml


* RWO PVC for stateful datasets (databases) 100GB (but only for extreme)
** For Backup: Create consistent snapshot, have plenty of time to do backup data
** As insurance: Do snap before change (e.g. application upgrade), have instant access to old data. = git versioning for your data
** As time machine: Messed all data up (e.g ransom ware)? Warp you data back in time to a good state
* Cloning
** instantly create up to date r/w copies of big data sets. Think CI/CD, DEV/QA
