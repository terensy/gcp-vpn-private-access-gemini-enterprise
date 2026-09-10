[繁體中文](README.md) | **English**

# How to Simulate an On-Premises Network in GCP and Access Gemini Enterprise Securely over a Private VPN Connection (A Private Service Connect + VPC Service Controls Walkthrough)

## TL;DR

When a real on-premises environment isn't available, you can build two VPCs in GCP — `sim-onprem-vpc`, which simulates the on-prem side, and `cloud-host-vpc`, which represents the cloud side — connected via **Cloud VPN**. On the cloud-side VPC, set up **Private Service Connect (PSC)** pointing at the Google APIs, so the simulated on-prem environment reaches **Gemini Enterprise** entirely over the private connection, without ever touching the public internet. Layer on **VPC Service Controls (VPC SC)** and **Access Context Manager (ACM)** to restrict the allowed source IPs, configure an Org Policy to limit the permitted data connector sources, and finally connect **Microsoft OneDrive** (via an Entra ID app registration and federated credentials) as the data source for the Gemini Enterprise DataStore.

Best suited for: enterprises that need to validate a "private VPN access only, no public internet access" security requirement before rolling out Gemini Enterprise, but don't yet have a physical on-premises network available for testing.

![architecture diagram](images/architecture-diagram.drawio.png)

## Table of Contents

- [TL;DR](#tldr)
- [1. Set Up sim-onprem-vpc to Simulate an On-Prem Network](#1-set-up-sim-onprem-vpc-to-simulate-an-on-prem-network)
  - [1.1 Create an On-Prem Subnet and a Test VM](#11-create-an-on-prem-subnet-and-a-test-vm)
  - [1.2 Configure the sim-onprem-vpc Firewall](#12-configure-the-sim-onprem-vpc-firewall)
  - [1.3 Set Up Cloud NAT (Public NAT)](#13-set-up-cloud-nat-public-nat)
  - [1.4 Set Up the VPN Gateway & Tunnel (HA VPN Is Optional for This Lab)](#14-set-up-the-vpn-gateway--tunnel-ha-vpn-is-optional-for-this-lab)
- [2. Set Up cloud-host-vpc as the GCP Cloud-Side Network](#2-set-up-cloud-host-vpc-as-the-gcp-cloud-side-network)
  - [2.1 Set Up Private Service Connect (PSC) Targeting All Google APIs](#21-set-up-private-service-connect-psc-targeting-all-google-apis)
  - [2.2 Set Up the VPN Gateway & Tunnel and Advertise the PSC IP](#22-set-up-the-vpn-gateway--tunnel-and-advertise-the-psc-ip)
- [3. Create a Dedicated GCP Project for Gemini Enterprise and Enable It](#3-create-a-dedicated-gcp-project-for-gemini-enterprise-and-enable-it)
  - [3.1 Gemini Enterprise Needs Workforce Identity Set Up Before the Dedicated Sign-In URL Appears](#31-gemini-enterprise-needs-workforce-identity-set-up-before-the-dedicated-sign-in-url-appears)
- [4. Configure VPC Service Controls (VPC SC) (Optional)](#4-configure-vpc-service-controls-vpc-sc-optional)
  - [4.1 Restricting Source IPs Requires Access Context Manager (ACM)](#41-restricting-source-ips-requires-access-context-manager-acm)
  - [4.2 Configure the GCP Projects and APIs to Protect](#42-configure-the-gcp-projects-and-apis-to-protect)
- [5. Update the Project-Level Org Policy](#5-update-the-project-level-org-policy)
  - [5.1 Enable the Data Connector Source Restriction](#51-enable-the-data-connector-source-restriction)
- [6. Create a Gemini Enterprise Datastore](#6-create-a-gemini-enterprise-datastore)
  - [6.1 Create the DataStore](#61-create-the-datastore)
  - [6.2 Connect the DataStore to OneDrive](#62-connect-the-datastore-to-onedrive)
- [7. Testing](#7-testing)
  - [7.1 Verify the Private Connection and Sign-In Flow](#71-verify-the-private-connection-and-sign-in-flow)
  - [7.2 Verify Functionality](#72-verify-functionality)
- [8. Route Gemini Enterprise Logs to Pub/Sub via the Cloud Logging Router](#8-route-gemini-enterprise-logs-to-pubsub-via-the-cloud-logging-router)
  - [8.1 Confirm Logging Is Enabled on the Gemini Enterprise App](#81-confirm-logging-is-enabled-on-the-gemini-enterprise-app)
  - [8.2 Create the Pub/Sub Service](#82-create-the-pubsub-service)
  - [8.3 Add a Log Router Sink in Cloud Logging](#83-add-a-log-router-sink-in-cloud-logging)
- [9. Consume the Pub/Sub Log Data with Open-Source ELK](#9-consume-the-pubsub-log-data-with-open-source-elk)
  - [9.1 Build the telemetry-demo-server VM and Install the ELK Stack](#91-build-the-telemetry-demo-server-vm-and-install-the-elk-stack)
  - [9.2 Configure a Logstash Pipeline to Subscribe to Pub/Sub](#92-configure-a-logstash-pipeline-to-subscribe-to-pubsub)
  - [9.3 Verify the Data and Explore It in Kibana](#93-verify-the-data-and-explore-it-in-kibana)
- [Keywords](#keywords)

---

## 1. Set Up sim-onprem-vpc to Simulate an On-Prem Network

### 1.1 Create an On-Prem Subnet and a Test VM

In `sim-onprem-vpc`, configure a subnet to represent an on-prem network segment, and spin up a Windows VM in that subnet for later testing. The VM should have no external IP; you'll sign in to it via IAP.

### 1.2 Configure the sim-onprem-vpc Firewall

- Add a firewall rule: allow ingress from `35.235.240.0/20` on TCP `3389`
- Add a firewall policy, and add the following rules to it:
  - Allow egress to FQDNs `discoveryengine.clients6.google.com`, `accounts.google.com.tw`, `lh3.google.com`, `lh3.googleusercontent.com`, `play.google.com` (these FQDNs can't be resolved to the PSC IP, so they need to go out over the public internet)
  - Allow egress to FQDNs `login.microsoftonline.com`, `aadcdn.msftauth.net`, `msauth.net`, `mysignins.microsoft.com` (only needed if you're using Entra ID SSO)
  - Deny egress to `0.0.0.0/0`

![firewall policy](images/firewall-policy-screenshot.png)

### 1.3 Set Up Cloud NAT (Public NAT)

The VM has no external IP, so the FQDNs allowed for egress in 1.2 (`discoveryengine.clients6.google.com` and the other domains that can't resolve to the PSC IP and need to go out over the public internet) can't actually reach the internet without a NAT gateway. You need to set up Cloud NAT (Public NAT) in `sim-onprem-vpc` and include the subnet's range in the NAT's source range, so those allowed egress rules have an actual path out to the internet.

### 1.4 Set Up the VPN Gateway & Tunnel (HA VPN Is Optional for This Lab)

> Notes:
> 1. When you build a pure cloud-to-cloud VPN session in GCP, set up one side first — you'll get a "Cloud Router BGP IP" and a "peer BGP IP." Use those two IPs to build the other side of the VPN.
> 2. Advertise the VM's subnet (or the VM's /32 IP) to the VPN.

## 2. Set Up cloud-host-vpc as the GCP Cloud-Side Network

### 2.1 Set Up Private Service Connect (PSC) Targeting All Google APIs

![private service connect](images/private-service-connect.png)

### 2.2 Set Up the VPN Gateway & Tunnel and Advertise the PSC IP

![cloud router advertise PSC IP](images/cloud-router-advertise-psc-ip.png)

## 3. Create a Dedicated GCP Project for Gemini Enterprise and Enable It

### 3.1 Gemini Enterprise Needs Workforce Identity Set Up Before the Dedicated Sign-In URL Appears

## 4. Configure VPC Service Controls (VPC SC) (Optional)

> This section is optional — it isn't required to validate the private connection in this lab. Build it out if your organisation needs additional control over the source IPs allowed to call the APIs.

### 4.1 Restricting Source IPs Requires Access Context Manager (ACM)

### 4.2 Configure the GCP Projects and APIs to Protect

## 5. Update the Project-Level Org Policy

### 5.1 Enable the Data Connector Source Restriction

Enable the "Restrict allowed data sources for data connectors" Org Policy and override the inherited (parent) setting, setting `allowedDataSources` to `onedrive` or whichever data source you need.

![org policy restrict allowed data sources](images/org-policy-Restrict-allowed-data-sources-for-data-connectors.png)

## 6. Create a Gemini Enterprise Datastore

### 6.1 Create the DataStore

You need to create a DataStore before Gemini Enterprise has any analytics data to show.

### 6.2 Connect the DataStore to OneDrive

1. First, create an application (an app registration) in Entra ID. Once it's created you'll get an "Application (client) ID" and a "Secret key value"; you'll also need the "Tenant ID". The app needs two redirect URLs configured.
   Reference: [entra-app-registration](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-onedrive/third-party-config#entra-app-registration)
2. Back in Gemini Enterprise, go to where you create the DataStore — mostly just follow the official documentation. For the connector mode, we recommend choosing "Federated Search".
   Reference: [set-up-data-store](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-onedrive/set-up-data-store)
3. Once the DataStore is created, you'll get a "Collection ID" — note it down, then go back to Entra ID to configure the "Federated credentials" again.

   ![gemini enterprise onedrive connector collection id](images/gemini-enterprise-onedrive-connector-collection-id.png)

4. Go to "Certificates & secrets" → "Federated credentials" → Add credential. Fill in the fields per the documentation below, and put the "Collection ID" in the Subject identifier field.
   Reference: [add-fed-credentials](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/ms-onedrive/third-party-config#add-fed-credentials)

   ![entra id federated credentials](images/entra-id-Federated-credentials.png)

5. Under API Permissions, enable `Files.Read.All`, `Sites.Read.All`, and `User.Read.All`. If you want to allow Gemini Enterprise to create, modify, or delete files, you'll also need `Files.ReadWrite.AppFolder` and `Files.ReadWrite`.

   ![entra id api permissions](images/entra-id-api-permissions.png)

## 7. Testing

### 7.1 Verify the Private Connection and Sign-In Flow

Sign in to the Windows VM, open a browser, and confirm you can't reach the internet. Then navigate to the Gemini Enterprise URL — you'll go through Google sign-in and Entra ID SSO along the way.

### 7.2 Verify Functionality

Confirm you can chat with the AI on the conversation page, then confirm you can search for documents stored in OneDrive.

## 8. Route Gemini Enterprise Logs to Pub/Sub via the Cloud Logging Router

### 8.1 Confirm Logging Is Enabled on the Gemini Enterprise App

![GE enable log](images/gemini-enterprise-enable-log.png)

### 8.2 Create the Pub/Sub Service

To better reflect a realistic enterprise landing zone resource hierarchy, this lab creates a separate, dedicated Telemetry project to host the Pub/Sub service.

### 8.3 Add a Log Router Sink in Cloud Logging

The key part is the filter — decide up front which log entries actually need to land in Pub/Sub. Skipping this filtering step will drive up your data transfer-out costs.

![Log Router Selector](images/log-router-selector.png)

## 9. Consume the Pub/Sub Log Data with Open-Source ELK

### 9.1 Build the telemetry-demo-server VM and Install the ELK Stack

Add another subnet to `sim-onprem-vpc` and create an Ubuntu 22.04 VM (`telemetry-demo-server`, again with no external IP, accessed via IAP). It lives in the same project and the same VPC as `sim-onprem-win-vm` from section 1.1, just in a different subnet. After adding Elastic's official APT repository, install Elasticsearch, Logstash, and Kibana directly through the package manager (version 8.19.21 at the time of writing) — all three services run on this single VM:

- Elasticsearch: listens on `9200` (HTTP) and `9300` (node-to-node)
- Logstash: listens on `5044` (Beats input, for reference) and `9600` (monitoring API)
- Kibana: listens on `5601`

> This is a stripped-down topology for lab validation. In production, split the three components across separate nodes based on data volume, and plan for a proper Elasticsearch cluster.

### 9.2 Configure a Logstash Pipeline to Subscribe to Pub/Sub

Install the official Logstash plugin `logstash-input-google_pubsub`, and add a pipeline config file under `/etc/logstash/conf.d/` that subscribes to the Pub/Sub subscription created in section 8:

```
input {
  google_pubsub {
    project_id    => "<your Telemetry project ID>"
    topic         => "gemini-enterprise-log-collector"
    subscription  => "gemini-enterprise-log-collector-sub"
    codec         => "json"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "gcp-logs-%{+YYYY.MM.dd}"
  }
}
```

The VM authenticates using the default Compute Engine service account (with the `cloud-platform` scope) via Application Default Credentials, so there's no need to download a separate key file. Just remember to grant that service account the `Pub/Sub Subscriber` role in the project where the subscription lives — otherwise Logstash won't actually be able to pull messages.

### 9.3 Verify the Data and Explore It in Kibana

Once the pipeline is running, use the following command to confirm new indices keep appearing and document counts keep growing:

```
curl -s localhost:9200/_cat/indices?v
```

You'll see indices split by date, e.g. `gcp-logs-2026.09.09`. Then, in Kibana (`http://<VM internal IP>:5601`), create the corresponding Data View so you can view, search, and visualise the usage records exported from Gemini Enterprise (query text, reply status, the API method invoked, and so on).

> ⚠️ On this lab VM, `xpack.security.enabled` is currently set to `false` — neither Elasticsearch nor Kibana has username/password authentication, and isolation relies solely on the VM having no external IP and being reachable only via IAP/inside the VPC. Before going to production, make sure to turn on Elastic's built-in security features (username/password or SSO) and add TLS; don't rely on network isolation alone.

## Keywords

GCP, Google Cloud, Gemini Enterprise, Cloud VPN, Private Service Connect (PSC), VPC Service Controls (VPC SC), Access Context Manager (ACM), Org Policy, Microsoft Entra ID, OneDrive Connector, Federated Credentials, SSO, Cloud Logging, Log Router, Pub/Sub, ELK, Elasticsearch, Logstash, Kibana, IAP, private connectivity, simulated on-premises network, hybrid cloud architecture
