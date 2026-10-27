# OpenShift on VMware vSphere — End-to-End Home Lab

Hands-on deployment of **Red Hat OpenShift Container Platform on VMware vSphere**, covering automated cluster provisioning, CoreOS, DNS, vSphere CSI storage, the internal image registry, and NGINX application deployment and troubleshooting.

> **Public portfolio note:** Hostnames, IP addresses, network names and infrastructure labels in this repository are intentionally anonymized. Documentation uses RFC 5737 TEST-NET address space and the reserved `.example` domain; these values do not represent the underlying environment.

> **What was actually built:** A **three-node compact OpenShift cluster**, with three RHCOS control-plane nodes also scheduling application workloads, on VMware vSphere. The cluster operators reported **OpenShift 4.22.18** (although I originally browsed the 4.20 installation materials). The completed installer run took **39 minutes 43 seconds**. A **six-node, Active Directory DNS/DHCP enterprise variant** is described at the end as a possible extension; it was not the topology deployed in this lab.
>
> This is a **non-production learning environment**. Addresses and names are lab examples. Never publish pull secrets, vCenter passwords, kubeconfigs, kubeadmin passwords, or installer state files.

## Architecture

~~~text
                         Windows / Ubuntu clients
                                  |
                          Home LAN 192.0.2.0/24
                                  |
                +-----------------+------------------+
                |                                    |
     Ubuntu infra-dns01                         VMware vCenter
     BIND9 DNS: .214                          vSphere / ESXi 8
     External DNS forwarding                         |
                |                         LAB-CLUSTER / datastore1
                |                                    |
                +---- API VIP: .60 -------------------+
                +---- Ingress VIP: .61 ---------------+
                                                     |
                     +-------------------------------+------------------------------+
                     |                               |                              |
                RHCOS master-0                  RHCOS master-1                 RHCOS master-2
                control-plane                   control-plane                  control-plane
                + worker                       + worker                       + worker
                     |                               |                              |
                     +--------- OVN-Kubernetes, workloads, registry, CSI ----------+
                                                          |
                                         OpenShift Route -> Service -> NGINX Pod
~~~

### Lab inventory

| Component | Actual configuration / observed value |
| --- | --- |
| Platform | VMware ESXi / vCenter, installer-provisioned vSphere (IPI) |
| OpenShift release | **4.22.18**, confirmed by cluster operators |
| Kubernetes | **v1.35.8**, confirmed by node output |
| Node topology | **3 control-plane nodes, 0 dedicated workers**, masters schedulable |
| Node names | ocp-lab-master-0, ocp-lab-master-1, ocp-lab-master-2 |
| Planned per-node VM resources | 4 vCPU, 16 GiB RAM, 100 GB OS disk |
| Virtual disk provision | **Thin** after correcting a thick-provisioned attempt |
| Storage class | thin-csi (default), csi.vsphere.vmware.com |
| Cluster name / base domain | ocp / lab.example |
| DNS | BIND9 on Ubuntu, **192.0.2.53** |
| API / Ingress VIPs | **192.0.2.60 / 192.0.2.61** |
| Temporary bootstrap IP | 192.0.2.62 |
| Control-plane IPs | 192.0.2.63, .64, .65 |
| Gateway | 192.0.2.1 |
| vSphere network | LAB-NET |
| Application project | demo |
| Application | NGINX unprivileged, verified with HTTP 200 (nginx/1.30.6) |

**Note:** Per-node CPU/RAM/disk values reflect the requested installation settings; verify actual VM allocations in vCenter. The image registry PVC was configured as a lab exercise later, but no final Bound/Running output was captured in this walkthrough.

## 1. Prerequisites and planning

This installation needs a supported vCenter/ESXi version, a reachable VMware cluster and datastore, adequate CPU/RAM/disk, DNS, outbound registry access, a Red Hat pull secret, and an Ubuntu/Linux machine for running the OpenShift installer.

Our working installation used one ESXi host for all three control-plane VMs. This is a **single physical failure domain**; three master VMs on one host do **not** provide physical high availability.

We initially had **563 GB free** on the selected datastore. The first provisioning attempt produced **thick disks**, which rapidly reduced available space to **2.83 GB**. We stopped, destroyed the partial deployment using installer state, and redeployed with **thin disks**. Monitor datastore capacity throughout the install.

Check VMware inventory, ensure the chosen datastore is accessible from the host, and reserve all node/VIP addresses **outside DHCP scope** before continuing.

## 2. Download the OpenShift installer

From the [Red Hat OpenShift vSphere installation portal](https://console.redhat.com/openshift/install/vsphere/installer-provisioned), download the **Linux x86_64 installer**, the **pull secret**, and a compatible OpenShift client (oc).

The installer provisions the RHCOS VMs automatically; manually deploying an RHCOS OVA is **not required** for the IPI path. An OVA from an unrelated release is not interchangeable with the release image selected by the installer.

Example commands (adjust filenames/paths to your downloaded release):

~~~bash
tar -xzf openshift-install-linux.tar.gz
chmod +x openshift-install
./openshift-install version
# Install the matching oc client using the official OpenShift client archive.
oc version --client
~~~

Keep the pull secret private. Don't commit the generated install-config.yaml or the auth directory to Git.

## 3. Trust the VMware vCenter certificate

Our first attempt failed with:

~~~text
Post "https://<vcenter>/sdk": tls: failed to verify certificate:
x509: certificate signed by unknown authority
~~~

The vCenter VMCA root certificates can be downloaded from the trusted vCenter server and imported into Ubuntu's CA store:

~~~bash
curl -k -o /tmp/vcenter-certs.zip https://<VCENTER_ADDRESS>/certs/download.zip
mkdir -p /tmp/vcenter-certs
unzip -o /tmp/vcenter-certs.zip -d /tmp/vcenter-certs
find /tmp/vcenter-certs/certs/win -name '*.crt'

# Copy the appropriate trusted CA (verify its identity/fingerprint out-of-band first).
sudo cp /tmp/vcenter-certs/certs/win/<CA_FILE>.crt \
  /usr/local/share/ca-certificates/vcenter-ca.crt
sudo update-ca-certificates

curl -Iv https://<VCENTER_FQDN_OR_IP>/sdk
~~~

The '-k' above is **only** used to retrieve the CA from the intended vCenter. Do not disable verification in the installer. If certificate subject names do not include the vCenter IP, configure and use the correct vCenter FQDN.

## 4. Configure DNS for the lab

For IPI on this VM network, the installer and RHCOS nodes must resolve the cluster endpoints and external Red Hat image registries.

Required example records:

| Record | Type | Answer |
| --- | --- | --- |
| api.ocp.lab.example | A | 192.0.2.60 |
| api-int.ocp.lab.example | A | 192.0.2.60 |
| *.apps.ocp.lab.example | Wildcard A | 192.0.2.61 |

The home router DNS did not support creating custom records, so we installed **BIND9 on an existing Ubuntu server**, at 192.0.2.53. This server's IP must be stable and reachable by every node and client that uses the cluster.

~~~bash
sudo apt update
sudo apt install -y bind9 bind9-utils dnsutils
sudo systemctl status named --no-pager
~~~

Example file **/etc/bind/named.conf.local**:

~~~conf
zone "ocp.lab.example" {
    type master;
    file "/etc/bind/db.ocp.lab.example";
};
~~~

Example **/etc/bind/db.ocp.lab.example**:

~~~dns
$TTL 300
@ IN SOA ns1.ocp.lab.example. admin.ocp.lab.example. (
  2026101901 3600 900 604800 300
)
@       IN NS ns1.ocp.lab.example.
ns1     IN A  192.0.2.53
api     IN A  192.0.2.60
api-int IN A  192.0.2.60
*.apps  IN A  192.0.2.61
~~~

In **/etc/bind/named.conf.options**, set suitable upstream DNS forwarding inside the existing options block:

~~~conf
forwarders {
    192.0.2.1;
    1.1.1.1;
};
forward only;
~~~

Validate and restart:

~~~bash
sudo named-checkconf
sudo named-checkzone ocp.lab.example /etc/bind/db.ocp.lab.example
sudo systemctl restart named
dig @192.0.2.53 api.ocp.lab.example +short
dig @192.0.2.53 api-int.ocp.lab.example +short
dig @192.0.2.53 sample.apps.ocp.lab.example +short
dig @192.0.2.53 rhcos.mirror.openshift.com +short
~~~

We fixed a real installation failure where internal lookups worked but external RHCOS mirror resolution failed. After enabling external forwarding, Ubuntu's systemd-resolved still returned **No appropriate name servers or networks for name found**. For the active interface (ens33), the working **temporary** remedy was:

~~~bash
sudo resolvectl dns ens33 192.0.2.53
sudo resolvectl domain ens33 '~.'
sudo resolvectl default-route ens33 yes
sudo resolvectl flush-caches
resolvectl query rhcos.mirror.openshift.com
resolvectl query api.ocp.lab.example
~~~

These resolvectl changes may disappear on reboot; persist them through the Ubuntu network manager/Netplan before relying on the server long term. Avoid manually editing /etc/resolv.conf when systemd-resolved manages it.

Test the actual RHCOS object URL chosen by the installer, rather than the mirror site's root URL (which can legitimately return HTTP 404).

## 5. Prepare install-config.yaml and static networking

Generate an initial installer configuration:

~~~bash
mkdir -p ~/ocp-install
./openshift-install create install-config --dir ~/ocp-install
~~~

Select **vSphere**, your vCenter, the datacenter, compute cluster, datastore, VM network, and the API/Ingress VIPs. Enter the Red Hat pull secret interactively; **never publish the resulting file**. In this lab the selected VIPs were **192.0.2.60** and **192.0.2.61**, base domain **lab.example**, and cluster name **ocp**.

Important sanitized excerpts from our configuration:

~~~yaml
compute:
- name: worker
  replicas: 0
  platform: {}

controlPlane:
  name: master
  replicas: 3
  platform:
    vsphere:
      osDisk:
        diskSizeGB: 100

platform:
  vsphere:
    diskType: thin
    apiVIPs:
    - 192.0.2.60
    ingressVIPs:
    - 192.0.2.61
    hosts:
    - role: bootstrap
      networkDevice:
        ipAddrs: [192.0.2.62/24]
        gateway: 192.0.2.1
        nameservers: [192.0.2.53]
    - role: control-plane
      networkDevice:
        ipAddrs: [192.0.2.63/24]
        gateway: 192.0.2.1
        nameservers: [192.0.2.53]
    - role: control-plane
      networkDevice:
        ipAddrs: [192.0.2.64/24]
        gateway: 192.0.2.1
        nameservers: [192.0.2.53]
    - role: control-plane
      networkDevice:
        ipAddrs: [192.0.2.65/24]
        gateway: 192.0.2.1
        nameservers: [192.0.2.53]
~~~

**This excerpt is NOT a complete install-config.yaml.** The real file includes installer-generated fields (including platform failure domains, vCenters, networking, metadata, pull secret and other settings). Keep those existing fields when applying changes.

Important troubleshooting lessons from our actual install:

- Setting **platform.vsphere.osDisk** failed schema validation with **unknown field "osDisk"**. We moved the setting under **controlPlane.platform.vsphere.osDisk.diskSizeGB: 100**, which passed create-manifests validation.
- Thick virtual disks unexpectedly consumed datastore capacity. We configured **platform.vsphere.diskType: thin** for the replacement deployment.
- For the compact topology, **compute.replicas: 0** with **controlPlane.replicas: 3** enables master scheduling; generated manifests reported **MastersSchedulable: true**.
- Validate static IPs, DNS, gateway, VLAN/port-group connectivity, vCenter permissions, OS disk policy, and accessible datastore **before** creating VMs.

The installer consumes install-config.yaml when generating assets. Save a private backup **outside the installer directory** with restrictive permissions:

~~~bash
cp ~/ocp-install/install-config.yaml ~/install-config-private.yaml
chmod 600 ~/install-config-private.yaml
~~~

Validate on a copy without deploying:

~~~bash
mkdir -p ~/ocp-validate
cp ~/ocp-install/install-config.yaml ~/ocp-validate/
./openshift-install create manifests --dir ~/ocp-validate
~~~

## 6. Automated IPI provisioning

After all prerequisites pass:

~~~bash
./openshift-install create cluster --dir ~/ocp-install --log-level=info
~~~

The installer imports the compatible RHCOS VMware image, creates a temporary bootstrap VM and three masters, configures the control plane and waits for operators. The bootstrap VM is removed after bootstrap completes.

Our successful installation reported **39m 43s total**:

| Stage | Reported duration |
| --- | ---: |
| Infrastructure pre-provisioning | 28 seconds |
| Network infrastructure | 39 seconds |
| Machine provisioning | 5m 15s |
| Bootstrap complete | 21m 19s |
| Bootstrap destruction | 5 seconds |
| Cluster operators available | 8m 10s |
| Cluster operators stable | 3m 43s |

Follow logs from a second terminal:

~~~bash
tail -f ~/ocp-install/.openshift_install.log
~~~

Once kubeconfig is available:

~~~bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc get nodes -o wide
oc get clusteroperators
oc get clusterversion
oc get events -A --sort-by='.lastTimestamp' | tail -25
~~~

Observed final node status:

~~~text
ocp-lab-master-0   Ready   control-plane,master,worker   v1.35.8
ocp-lab-master-1   Ready   control-plane,master,worker   v1.35.8
ocp-lab-master-2   Ready   control-plane,master,worker   v1.35.8
~~~

Console, ingress and authentication cluster operators were **Available=True, Progressing=False, Degraded=False**, with version **4.22.18**.

The lab console used:

~~~text
https://console-openshift-console.apps.ocp.lab.example
~~~

Fresh lab installations may use an ingress certificate that isn't trusted by client machines. A temporary **connectivity-only test** is:

~~~bash
curl -kI https://console-openshift-console.apps.ocp.lab.example/
~~~

We verified **HTTP 200 OK**. In production, use an appropriate trusted ingress certificate. The kubeadmin password and kubeconfig are secrets: do not upload them.

## 7. Windows workstation access to Routes

Linux could resolve the OpenShift application hostname through BIND9, but Windows Chrome showed **DNS_PROBE_FINISHED_NXDOMAIN** because the Windows host used router DNS instead of BIND9.

Test:

~~~powershell
nslookup console-openshift-console.apps.ocp.lab.example 192.0.2.53
nslookup someapp-demo.apps.ocp.lab.example 192.0.2.53
~~~

For a permanent solution, configure client DNS to reach the authoritative records (for example the lab BIND9 server or a managed forwarding rule). For a short test, the Windows hosts file can map **individual** Route hostnames to **192.0.2.61**; hosts files do **not** implement DNS wildcards.

## 8. vSphere CSI storage and registry

Our cluster exposed:

~~~text
thin-csi (default)  csi.vsphere.vmware.com  Delete  WaitForFirstConsumer
~~~

The StorageClass referenced the vSphere storage policy **openshift-storage-policy-ocp-lab**. Always inspect the vCenter storage policy to confirm how it handles provisioning before requesting large volumes.

Useful commands:

~~~bash
oc get sc
oc get csidrivers
oc get co storage image-registry
oc get pvc -A
~~~

The image-registry operator initially reported **managementState: Removed**; this caused S2I to fail with:

~~~text
InvalidOutputReference: an image stream cannot be used as build output
because the integrated container image registry is not configured
~~~

For the PVC-backed registry lab, use **one registry replica**, **ReadWriteOnce**, **Recreate** rollout, and the vSphere CSI StorageClass. Example PVC:

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  storageClassName: thin-csi
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 100Gi
~~~

~~~bash
oc apply -f registry-pvc.yaml
oc patch configs.imageregistry.operator.openshift.io cluster --type=merge \
  -p '{"spec":{"managementState":"Managed","replicas":1,"rolloutStrategy":"Recreate","storage":{"pvc":{"claim":"image-registry-storage"}}}}'

oc get pvc -n openshift-image-registry
oc get pods -n openshift-image-registry
oc get co image-registry
~~~

**Verify the PVC is Bound and the registry pod is Ready before claiming this step is finished.** With WaitForFirstConsumer, the PVC can remain Pending until a registry pod is schedulable. This walkthrough documents the planned/applied setup, but the conversation did not capture final PVC/registry status output.

## 9. NGINX app: deployment, security and OpenShift Route

We created a deployment in namespace **demo**. The original NGINX image failed:

~~~text
CrashLoopBackOff
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
~~~

The stock image attempted to write to directories unavailable under OpenShift's restricted, non-root Security Context Constraints (SCC). Avoid solving this by indiscriminately granting privileged or anyuid SCC.

The fix was to use the **unprivileged NGINX image**:

~~~bash
oc project demo
oc set image deployment/demo-pod \
  nginx=nginxinc/nginx-unprivileged:stable -n demo
~~~

However, the Deployment also contained the OpenShift ImageStream trigger for **nginx:1.24**, which **overwrote the manual image update** and recreated the failing ReplicaSet.

We diagnosed it with:

~~~bash
oc get deployment demo-pod -n demo -o yaml
oc get rs -n demo
oc describe rs demo-pod-6774bcbb84 -n demo
oc logs deployment/demo-pod -n demo --tail=50
~~~

Two ReplicaSets illustrated the problem:

| ReplicaSet | Image | Result |
| --- | --- | --- |
| demo-pod-6774bcbb84 | Original nginx digest | Crashed with permission denied |
| demo-pod-86544c4 | nginxinc/nginx-unprivileged:stable | Running |

Disable the unneeded ImageStream trigger **before** setting the replacement image:

~~~bash
oc annotate deployment/demo-pod image.openshift.io/triggers- -n demo
oc set image deployment/demo-pod \
  nginx=nginxinc/nginx-unprivileged:stable -n demo
oc rollout status deployment/demo-pod -n demo
oc get pods,rs -n demo
~~~

The unprivileged image listens on **8080**. Our first Service had **targetPort 80** and selected a particular ReplicaSet via **pod-template-hash**, so it was vulnerable to port errors and updates.

Use a stable Deployment-level selector and forward the Service to 8080. Example:

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo
spec:
  selector:
    deployment: demo-pod
  ports:
  - name: http
    port: 80
    targetPort: 8080
~~~

~~~bash
oc apply -f nginx-service.yaml
oc expose svc/nginx-svc --name=nginx-route -n demo
oc get svc,route -n demo
oc get endpoints nginx-svc -n demo
curl -I "http://$(oc get route nginx-route -n demo -o jsonpath='{.spec.host}')"
~~~

The deployed NGINX app returned **HTTP/1.1 200 OK**, with **nginx/1.30.6** reported by its Server header. This validated **Pod -> Service -> OpenShift Route -> Ingress VIP -> client** connectivity. An OpenShift Route is **not** created automatically when you create a Kubernetes Service.

### S2I / ImageStream exercise

We also experimented with **Source-to-Image (S2I)** using the Git repository [sclorg/nginx-container](https://github.com/sclorg/nginx-container) and the OpenShift nginx:1.24 ImageStream. This is a separate workflow from deploying a prebuilt container image.

Our BuildConfig output targeted **ImageStreamTag nginxtest:latest**. The initial build stayed at **New (InvalidOutputReference)** while the internal image registry was **Removed**. We also learned to check that the S2I builder image is the right Red Hat/OpenShift-compatible image, rather than an ordinary NGINX runtime image. This walkthrough does **not** claim that the S2I build ultimately completed successfully.

Troubleshooting:

~~~bash
oc get bc,build,is -n demo
oc describe build nginxtest-1 -n demo
oc get configs.imageregistry.operator.openshift.io cluster -o yaml
~~~

## 10. Troubleshooting summary

| Symptom | Cause found / investigation | Resolution |
| --- | --- | --- |
| vCenter x509 unknown authority | VMCA cert not trusted by installer Ubuntu | Import verified vCenter CA into Ubuntu trust |
| RHCOS OVA download: server misbehaving | Broken external name resolution on installer | BIND9 forwarders; systemd-resolved DNS routing |
| Datastore nearly full | vSphere VM disks thick provisioned | Stop, installer destroy, fresh install with thin disk type |
| Installer JSON unknown field osDisk | Field placed under platform.vsphere | Move to controlPlane.platform.vsphere.osDisk |
| OpenShift console curl certificate error | Default ingress certificate not trusted | Confirm HTTP with curl -k, configure trusted ingress cert later |
| S2I InvalidOutputReference | Internal image registry managementState Removed | Provision suitable storage, enable registry |
| NGINX CrashLoopBackOff | Non-root UID denied write to NGINX cache | Use nginx-unprivileged image |
| Deployment reverted image | ImageStream trigger repeatedly selected original image | Remove trigger and update Deployment |
| Route not listed | Only Service created | oc expose svc |
| Service not serving app | targetPort 80 vs container 8080; hash-specific selector | Fix Service port and stable selector |
| Linux curl works, Windows Chrome NXDOMAIN | Windows not using DNS containing wildcard apps record | Configure client DNS or temporary hosts entry |
| Pull image TLS bad record MAC | Download transport/TLS integrity error; cause not conclusively determined | Investigated retry/network/MTU; do not claim proven root cause |

## 11. Useful validation commands

~~~bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc get nodes -o wide
oc get clusterversion
oc get co
oc get pods -A
oc get sc
oc get pvc -A
oc get builds -n demo
oc get deployment,rs,pod,svc,route -n demo
oc describe pod <POD_NAME> -n demo
oc logs deployment/demo-pod -n demo
~~~

## 12. Enterprise extension — not part of this deployment

A production-style design can separate roles into **three control-plane and three worker nodes**, spread across multiple physical hosts or failure domains, and use enterprise **Microsoft AD-integrated DNS** and **Windows Server DHCP** (or IPAM/DHCP platforms such as Infoblox).

| Enterprise service | Example responsibility |
| --- | --- |
| AD-integrated DNS | api, api-int, wildcard *.apps records; redundant name servers |
| Enterprise DHCP/IPAM | Reserved addresses and Option 006 DNS servers; Option 003 gateway where appropriate |
| Load balancing | Supported resilient API/Ingress VIP services where the topology requires them |
| VMware | Multiple ESXi hosts, storage policy, fault domains and monitoring |
| Storage | vSphere CSI and appropriately designed persistent registry storage |
| Security | Dedicated vCenter service account, trusted certs, image policy, least privilege |

This six-node configuration is a **design/expansion example**, not a claim that six nodes or AD DNS/DHCP were used in the home lab.

## Security and cleanup

- Keep **install-config.yaml**, pull secrets, kubeconfig, kubeadmin password, vCenter credentials, Ignition files and installer state **out of public Git**.
- Rotate any administrative password exposed in a chat, terminal capture, or screenshot.
- Prefer a dedicated vCenter service account instead of the built-in administrator.
- To **destroy** a lab provisioned by this installer, retain the correct install directory/state and use:

~~~bash
./openshift-install destroy cluster --dir ~/ocp-install --log-level=info
~~~

This is destructive. Confirm the target cluster before running it. Do not manually remove provisioned VMDKs while the installer is managing them.

---

### References

- [Red Hat OpenShift vSphere installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_on_vmware_vsphere/)
- [Red Hat OpenShift installer downloads](https://console.redhat.com/openshift/install/vsphere/installer-provisioned)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [OpenShift Source-to-Image](https://github.com/openshift/source-to-image)

**Result:** A working three-node compact OpenShift-on-vSphere cluster, healthy control-plane nodes and operators, a functioning local DNS integration, a VMware CSI storage class, and a reachable NGINX deployment via an OpenShift Route.

## Architecture design documents

- [High-Level Design (HLD)](docs/HLD_OpenShift_on_VMware.md) — platform architecture, component roles, network/data flows, security and availability.
- [Low-Level Design (LLD)](docs/LLD_OpenShift_on_VMware.md) — VM and IP inventory, DNS, storage, OpenShift settings, application deployment and verification commands.

These document the **actual three-node compact home lab**. The six-node AD DNS/DHCP design is described separately as an **enterprise extension**, not as an implemented topology.
