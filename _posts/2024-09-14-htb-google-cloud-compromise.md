---
title: "HackTheBox | Google Cloud Compromise"
date: 2024-09-14 03:00:00 +0100
categories: [HackTheBox]
tags: [htb, sherlock, dfir, gcp, cloud, sof-elk, writeup]
image:
  path: https://github.com/user-attachments/assets/d77996f9-6a93-44ae-9eeb-add2e6b6f02c

---

# Summary

In this Sherlock, a startup stores its secret code in a Google Cloud Storage bucket, and a greedy developer steals it by impersonating another Google Cloud identity. We're given the project's Google Cloud logs and have to figure out what happened. Instead of reading the JSON by hand, I set up SOF-ELK, ingested the logs into it and solved every question from Kibana.

# What is SOF-ELK?

SOF-ELK (Security Operations and Forensics ELK) is a free, pre-configured VM built on the Elastic Stack and used in several SANS courses. You drop log files into a folder under `/logstash/`, and SOF-ELK parses them, enriches them (GeoIP, ASN, user agent) and makes them searchable in Kibana. It supports a lot of formats out of the box, including GCP logs exported with `gcloud`, which is exactly what HTB gives us.

# Setting up SOF-ELK

I downloaded the x86 build from [https://for572.com/sof-elk-x86-vm](https://for572.com/sof-elk-x86-vm) (there's also an ARM build for Apple Silicon) and verified it against the SHA256 hash published on the [SOF-ELK wiki](https://github.com/philhagen/sof-elk/wiki/Virtual-Machine-README):

`Get-FileHash "Public+SOF-ELK+x86+v20260706.7z" -Algorithm SHA256`

> <img width="1093" height="125" alt="IMG-01-hash-check" src="https://github.com/user-attachments/assets/635f7a48-5489-4325-a98f-34f9fe6d8192" />
 `Get-FileHash` output matching the published hash

After extracting it, we get a `.vmx` file (the VM's configuration: RAM, CPU, network) and a `.vmdk` file (the virtual disk with the OS and SOF-ELK itself). Opening the `.vmx` in VMware Workstation loads the VM, and we can log in with `elk_user` / `forensics`:

> <img width="686" height="598" alt="IMG-02-login-screen" src="https://github.com/user-attachments/assets/5d8490c3-38bc-4681-a459-055a1b05970a" />


Running `ip -4 addr` gives us the VM's IP, and Kibana is then reachable from the host at `http://<vm-ip>:5601`:

> <img width="1895" height="956" alt="IMG-03-kibana-home" src="https://github.com/user-attachments/assets/a563521c-bb16-460f-8f3e-915fc67e7e3d" />
 SOF-ELK introduction dashboard in Kibana

# Ingesting the logs

The Sherlock gives us `gcp.json`, a JSON array of 6 Cloud Audit Log entries. SOF-ELK watches its folders for new files, so ingesting is just copying the file into `/logstash/gcp/`. I did it from my host with `scp`:

`scp .\gcp.json elk_user@192.168.178.128:/logstash/gcp/`

> <img width="1056" height="219" alt="IMG-04-scp-transfer" src="https://github.com/user-attachments/assets/304b1d90-765d-47ff-94e8-4700dd599aab" />
 `scp` transfer in PowerShell

To make sure it parsed correctly, I asked Elasticsearch how many documents it has in the GCP index:

`curl -s 'localhost:9200/gcp-*/_count?pretty'`

> <img width="547" height="297" alt="IMG-05-count-6" src="https://github.com/user-attachments/assets/8cebdee4-2bb4-483f-9383-bf0b6d582752" />
 

6 documents for 6 log entries, so everything was ingested.

# Building the investigation view

In Kibana's **Discover**, I selected the `gcp-*` data view. At first nothing shows up, because the default time range is the last 15 minutes and these logs are from July 2023. After setting the time range to cover July 2023, we see our 6 events, but each one is a wall of text:

> <img width="1904" height="953" alt="IMG-07-discover-raw" src="https://github.com/user-attachments/assets/aa45e959-c4b2-48f9-a1a9-8ede50a395ee" />
 Discover showing the 6 raw documents

SOF-ELK maps the GCP fields to cleaner names, so I built a table around the questions every investigation needs to answer:

| Column | Question | What it describes |
|---|---|---|
| `@timestamp` | When? | When the event happened (sorted oldest first) |
| `user.name` | Who? | The identity that made the request (`principalEmail`) |
| `gcp.audit.method_name` | What? | The API call that was made (`methodName`) |
| `source.ip` | From where? | The IP the request came from (`callerIp`) |
| `user_agent.original` | With what? | The tool, OS and command used (`callerSuppliedUserAgent`) |
| `gcp.audit.request.resource_name` | To what? | The bucket, file or service targeted (`resourceName`) |

> <img width="1912" height="904" alt="IMG-08-discover-table" src="https://github.com/user-attachments/assets/55708a3c-b64e-4342-90f2-4eb880be62f7" />
 The finished Discover table

Not every GCP log has these fields: only Cloud Audit Logs record who did what and from where. Even then, some can be empty. The `storage.buckets.list` event, for example, has no resource name because it doesn't target a single bucket.

I also noticed a few gaps in SOF-ELK's GCP parsing: the error status code isn't extracted into a field, the user agent parser labels the device as `Other`, and the GeoIP database in the VM is from 2019. The raw log is always kept in `event.original`, so nothing is lost. I saved the view so I can reopen it later.

# Investigation

## What's the name of the project that was compromised?

My first guess was the long number in `projects/421111031168/services/...`, but that's the **project number**, an ID Google assigns automatically. What we want is the **project ID**, the human-readable identifier chosen when the project is created. It's in the `gcp.instance.project_id` field:

> <img width="210" height="130" alt="IMG-09-project-id" src="https://github.com/user-attachments/assets/d0840665-b06e-4038-8b44-a2adc500a948" />

**Answer:** `cryptostartup`

## What google cloud identity is compromised?

The `user.name` field shows the same service account on every event:

> <img width="216" height="169" alt="IMG-10-user-name" src="https://github.com/user-attachments/assets/d3c23c93-9a12-48e9-bc1f-f2c8a723dd45" />

Looking at `event.original`, every request includes a `serviceAccountKeyName`, so the attacker authenticated using a downloaded key for this service account.

**Answer:** `cloud-storage-helper@cryptostartup.iam.gserviceaccount.com`

## What IP address is the identity authenticated from?

All 6 requests come from the same IP in `source.ip`:

> <img width="157" height="102" alt="IMG-11-source-ip" src="https://github.com/user-attachments/assets/5d366a27-dbad-413a-b280-a1868a7dd417" />

**Answer:** `178.132.108.38`

## What country does the IP originate from?

SOF-ELK's GeoIP enrichment says Italy, but since its database is from 2019, I checked the IP on [ipinfo.io](https://ipinfo.io):

> <img width="1107" height="907" alt="IMG-12-ipinfo" src="https://github.com/user-attachments/assets/77247920-2cba-4ce8-9518-f5c2c4aa75be" />
 ipinfo.io result

ipinfo shows Bucharest, Romania, and flags the IP as a VPN/hosting provider. The IP block has clearly changed owners since SOF-ELK's database was built, which is a good reminder to always confirm enrichment with a current source.

**Answer:** `Romania`

## What device did the attack come from?

Google doesn't log devices directly, but the user agent in `user_agent.original` gives it away:

> <img width="214" height="264" alt="IMG-13-user-agent-device" src="https://github.com/user-attachments/assets/48e35e85-a040-4e8f-a23d-36463a786cf3" />

SOF-ELK's parsed device field just says `Other` here, so the raw user agent is the one to read.

**Answer:** `Macintosh`

## What was the first failed API call made by this identity?

The first event in the timeline, which failed with a permission denied error:

> <img width="220" height="98" alt="IMG-14-enableservice" src="https://github.com/user-attachments/assets/8035d61b-d1bd-41a5-8e50-cf2fc20341b3" />

The user agent shows the attacker was actually running `gcloud compute firewall-rules list`. The Compute API wasn't enabled in the project, so `gcloud` tried to enable it first, and the service account didn't have permission. The next two events are the same story with `firewall-rules create` and `projects get-iam-policy`.

**Answer:** `google.api.serviceusage.v1.ServiceUsage.EnableService`

## What storage bucket was enumerated?

After failing with Compute, the attacker moved to storage. At `00:17:18`, there's a `storage.objects.list` call on `projects/_/buckets/importantbucket`, and the user agent shows `command/ls`, so they listed the bucket's contents:

> <img width="631" height="117" alt="IMG-15-bucket-enum" src="https://github.com/user-attachments/assets/82b0f106-d790-4e94-b649-967966f8f5d7" />

**Answer:** `importantbucket`

## What API call was used to exfiltrate an item from this bucket?

The last event in the timeline:

> <img width="221" height="86" alt="IMG-16-objects-get" src="https://github.com/user-attachments/assets/35debd2d-ccb0-4442-8df5-e974ddae46f5" />

`storage.objects.get` means an object was downloaded from the bucket.

**Answer:** `storage.objects.get`

## Which Google Cloud command-line tool was used during the exfiltration attempt?

The user agent of that same event shows `gsutil/5.10 ... command/cp`, so the attacker used `gsutil cp` to copy the file to their machine:

> <img width="1069" height="118" alt="IMG-17-gsutil-cp" src="https://github.com/user-attachments/assets/4f633acc-86be-41f3-94bc-253719dc06bb" />
 `user_agent.original` of the `storage.objects.get` event

**Answer:** `gsutil`

## What is the name of the file that was exfiltrated from the storage bucket?

The `gcp.audit.request.resource_name` of the last event:

> <img width="281" height="116" alt="IMG-18-resource-name" src="https://github.com/user-attachments/assets/a6d93f41-7cf0-4822-8107-4469a15143bb" />

**Answer:** `secretcode.java`

# Timeline

| Time (UTC) | Command | Method | Result |
|---|---|---|---|
| 00:16:24 | `gcloud compute firewall-rules list` | `EnableService` | Denied |
| 00:16:43 | `gcloud compute firewall-rules create` | `EnableService` | Denied |
| 00:17:03 | `gcloud projects get-iam-policy` | `EnableService` | Denied |
| 00:17:11 | `gsutil ls` | `storage.buckets.list` | Success |
| 00:17:18 | `gsutil ls` | `storage.objects.list` | Success |
| 00:17:36 | `gsutil cp` | `storage.objects.get` | Success |

The whole thing took just over a minute: the attacker tried to touch the firewall and IAM, got blocked, then went straight for the bucket and grabbed the secret code.

