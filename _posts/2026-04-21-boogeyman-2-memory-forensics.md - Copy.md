---
layout: post
title: "SIEM & Memory Forensics: Boogeyman 2 Triage"
date: 2026-04-21 00:00:00 +0200
categories: ["Threat Hunting & Triage", "SIEM Investigations"]
tags: [volatility, memory-forensics, c2-detection, incident-response, blue-team]
pin: true
image:
  path: https://tryhackme-images.s3.amazonaws.com/room-icons/2fb0a9b7f813718a338cbc765071c7e7.png
  # This hides it from the internal post header in almost all Chirpy versions
  preview: false
  # If the thumbnail is cropped, try adding this class
  class: img-contain
---

# Alert Triage With Splunk

![Alert Triage With Splunk Banner](https://github.com/user-attachments/assets/65dcd63a-87c5-4f9c-9613-d960c5b64cc2
){: .shadow .rounded }
_Figure 1: [Room Name] Challenge overview._

## Scenario

> [Insert the scenario or room description from the platform here.]
{: .prompt-warning }

![Scenario or Alert details]([Optional_Scenario_Image_URL]){: .shadow .rounded }
_Figure 2: Alerts triggering the investigation._

---

## Phase 1: [Name of Investigation Phase]

> **Question:** [Insert Question 1 Here]
{: .prompt-info }

[Insert explanation text here. Explain your thought process and what tools you used.]

```bash
[Insert command line code here, if applicable]
```
{: .nolineno }

![[Image description]]([Insert_Step_Image_URL_Here]){: .shadow .rounded }

> **Answer:** `[Insert Answer Here]`
{: .prompt-tip }
<br>

---

> **Question:** [Insert Question 2 Here]
{: .prompt-info }

[Insert explanation text here.]

![[Image description]]([Insert_Step_Image_URL_Here]){: .shadow .rounded }

> **Answer:** `[Insert Answer Here]`
{: .prompt-tip }
<br>

---

## Phase 2: [Name of Next Phase]

> **Question:** [Insert Question 3 Here]
{: .prompt-info }

[Insert explanation text here.]

```bash
[Insert command line code here]
```
{: .nolineno }

![[Image description]]([Insert_Step_Image_URL_Here]){: .shadow .rounded }

> **Answer:** `[Insert Answer Here]`
{: .prompt-tip }
<br>

---

<style>
  /* Hides the top banner inside the post */
  #post-wrapper > img:first-of-type,
  .preview-img,
  header + img {
    display: none !important;
  }
</style>

<style>
  /* 1. Fix the "Outside" (Home Page Card) - prevents cropping */
  .post-preview .preview-img img {
    object-fit: contain !important;
    background-color: #1b1b1e; /* Matches Chirpy's dark background */
  }

  /* 2. Fix the "Inside" - hides the banner completely */
  #post-wrapper img.preview-img,
  header + .preview-img,
  .post-content > img:first-child {
    display: none !important;
  }
</style>
