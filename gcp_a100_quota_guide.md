# Guide: Requesting NVIDIA A100 GPU Quota on Google Cloud Platform

This document outlines the necessary quota adjustments required to successfully deploy NVIDIA A100 GPU instances (regular or preemptible) on Google Cloud Platform (GCP).

---

## 1. Core Quota Attributes to Adjust

To run an A100 GPU instance (such as the smallest `a2-highgpu-1g` machine type), you must request and receive approval for **three distinct quotas**. Failing to increase any of these will result in deployment failures.

### A. Global GPU Limit (Project-wide)
* **Metric Name:** `compute.googleapis.com/gpus_all_regions` (GPU_ALL_REGIONS)
* **Scope:** Global (Project-wide)
* **Purpose:** Sets the total cap on the number of GPUs of any model that can be running simultaneously across all regions in your project.
* **Required Value:** Minimum **`1`** (or the total number of GPUs you plan to run concurrently).

### B. Regional A100 GPU Limit
* **Metric Name:** 
  * For Standard: `compute.googleapis.com/nvidia_a100_gpus` (NVIDIA A100 GPUs)
  * For Preemptible: `compute.googleapis.com/preemptible_nvidia_a100_gpus` (Preemptible NVIDIA A100 GPUs)
* **Scope:** Regional (e.g., `asia-northeast1`)
* **Purpose:** Authorizes the allocation of the specific A100 GPU hardware in your chosen region.
* **Required Value:** Minimum **`1`**.

### C. Regional CPU Limit
* **Metric Name:** 
  * For Standard: `compute.googleapis.com/cpus` (CPUs)
  * For Preemptible: `compute.googleapis.com/preemptible_cpus` (Preemptible CPUs)
* **Scope:** Regional (must match your GPU region)
* **Purpose:** Every GPU machine type requires a minimum number of host vCPUs. The smallest instance type, `a2-highgpu-1g` (1 A100), requires **12 vCPUs**. If your regional CPU quota is 0, the instance cannot be created.
* **Required Value:** Minimum **`12`** (or more depending on the machine type scale).

---

## 2. Step-by-Step Request Instructions

1. Open the [Google Cloud Console](https://console.cloud.google.com/).
2. Select your target project from the project dropdown list at the top.
3. In the left navigation menu, go to **IAM & Admin > Quotas & System Limits**.
4. Use the filter bar to search for and select the required metrics:
   * **`compute.googleapis.com/gpus_all_regions`**
   * **`compute.googleapis.com/preemptible_nvidia_a100_gpus`** (or standard)
   * **`compute.googleapis.com/preemptible_cpus`** (or standard)
5. For the regional metrics, filter by your target region (e.g., `asia-northeast1`).
6. Tick the checkbox next to the quota you want to increase, then click **EDIT QUOTAS** at the top of the page.
7. Enter the new limit values in the side panel, fill out the required reason/justification, and click **SUBMIT REQUEST**.

---

## Signature
* **Model:** Gemini 3.5 Flash
* **Timestamp:** 2026-05-23T16:42:14+08:00
