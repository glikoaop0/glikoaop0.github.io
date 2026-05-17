---
layout: ../../layouts/Article.astro
title: "GCP Unauthorized External Access: Hunt & Response"
date: 2026-05-02
tags: [Cloud, GCP, Threat Hunting]
excerpt: "How I identified an unauthorized principal on a client GCP project, built a hunting playbook, and bridged the finding into Splunk detections."
draft: true

---

During a routine cloud hunting session, I identified an unauthorized external principal with `roles/editor` access on a client's GCP project. Here's how the hunt unfolded and what detection logic came out of it.

## Initial Signal

The first hint came from Wiz — a medium-severity finding flagged an IAM binding where the principal type was `allAuthenticatedUsers`, something that should never appear in this client's environment.

```bash
# gcloud equivalent of what we saw in the audit log
gcloud projects get-iam-policy PROJECT_ID \
  --format="json" | jq '.bindings[] | select(.members[] | contains("allAuthenticatedUsers"))'
```

## Hunting in Splunk

After confirming the finding, I pivoted to Cloud Audit Logs ingested into Splunk to understand the blast radius:

```spl
index=gcp sourcetype="google:gcp:pubsub:message"
  protoPayload.methodName="SetIamPolicy"
  protoPayload.request.policy.bindings{}.members{}="allAuthenticatedUsers"
| table _time, protoPayload.authenticationInfo.principalEmail, protoPayload.resourceName
| sort -_time
```

The query surfaced the exact timestamp and principal that made the change — an external service account that had been granted editor rights 11 days prior.

## What to Look For

When hunting GCP IAM anomalies, focus on these patterns:

- Bindings added for `allUsers` or `allAuthenticatedUsers`
- New `roles/owner` or `roles/editor` grants to external domains
- Service account key creation followed by API calls from unexpected IPs
- `SetIamPolicy` calls during off-hours from unfamiliar principals

## Detection Rule

Promoted to Splunk notable alert after validation:

```spl
index=gcp sourcetype="google:gcp:pubsub:message"
  protoPayload.methodName="SetIamPolicy"
| eval members=mvexpand(protoPayload.request.policy.bindings{}.members{})
| where match(members, "allUsers|allAuthenticatedUsers|@external-domain\.com")
| stats count by protoPayload.authenticationInfo.principalEmail, protoPayload.resourceName, _time
```

## Takeaway

The key gap was no alerting on IAM policy changes. Wiz caught it on the configuration side, but there was no behavioral detection to catch the *act* of granting the permission. Both layers are necessary.
