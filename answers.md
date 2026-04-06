# API Ethics Assignment — Answers

**Name:** Om Akash  
**Role:** Junior Data Analyst (Healthcare Startup)

---

## Task 1 — Classify and Handle PII Fields

### What is PII?

**PII (Personally Identifiable Information)** is any data that can be used to identify a specific individual, either on its own or when combined with other information.

- **Direct PII** — directly and uniquely identifies a person on its own (e.g. full name, email)
- **Indirect PII** — does not identify someone alone, but can identify them when combined with other fields (e.g. zip code + date of birth together can narrow down a person)

---

### Field Classification Table

| Field | PII Type | Action | Justification |
|---|---|---|---|
| `full_name` | Direct PII | **Drop** | A person's name directly identifies them. Research partners don't need real names — a random ID is enough. |
| `email` | Direct PII | **Drop** | Email addresses uniquely identify individuals and can be used to contact them without consent. No research value. |
| `date_of_birth` | Indirect PII | **Mask** → store only the year (e.g. `1990`) | Exact birth dates combined with other fields can re-identify a person. Keeping just the year preserves research value (age groups) without the risk. |
| `zip_code` | Indirect PII | **Mask** → truncate to first 3 digits (e.g. `600` instead of `600032`) | Full zip codes can pinpoint a small neighbourhood. Truncating to a broader region reduces re-identification risk while keeping geographic usefulness. |
| `job_title` | Indirect PII | **Pseudonymize** → group into broad categories (e.g. "Healthcare", "Engineering", "Education") | Rare job titles (e.g. "Chief Neurosurgeon, Apollo Hospital Chennai") can identify a person. Grouping into broad categories keeps occupational data useful for research. |
| `diagnosis_notes` | Direct PII (Sensitive) | **Pseudonymize** → remove names/locations, keep medical terms only | Diagnosis notes may contain the patient's own words, names, and locations. Medical content is valuable to researchers, but all identifying references must be stripped out first. |

---

### Summary

Before sharing with the external research partner, the dataset should look like this:

| Original Field | Shared As |
|---|---|
| `full_name` | *(dropped — replaced by a random `patient_id`)* |
| `email` | *(dropped entirely)* |
| `date_of_birth` | `birth_year` (e.g. `1990`) |
| `zip_code` | `region_code` (first 3 digits only) |
| `job_title` | `occupation_group` (broad category) |
| `diagnosis_notes` | `diagnosis_notes` (cleaned — names/locations removed) |

---

## Task 2 — Audit the API Script for Ethical Compliance

### The Original Script

```python
import requests

API_URL = "https://healthstats-api.example.com/records"
API_KEY = "free_tier_key_abc123"

records = []
for page in range(1, 101):
    response = requests.get(API_URL, params={"page": page, "key": API_KEY})
    data = response.json()
    records.extend(data["results"])

# Store all records permanently in company database
save_to_database(records)
```

---

### Violation 1 — Collecting More Data Than Allowed (Exceeding API Limits)

**What is the problem?**

The script blindly fetches **100 pages** of records in a loop (`range(1, 101)`) without checking:
- Whether the API actually has that many pages
- Whether the API's Terms of Service (TOS) allow bulk downloading of this much data
- Whether a rate limit is being respected

Most free-tier APIs (like this one using `free_tier_key_abc123`) have strict limits on how many records you can pull. Scraping 100 pages in one go likely violates the TOS. It also puts unnecessary load on the API server. In a healthcare context, pulling thousands of patient records without a documented legal basis (like a data sharing agreement) may also violate privacy laws such as **HIPAA** or **GDPR**.

**Corrected Code:**

```python
import requests
import time

API_URL = "https://healthstats-api.example.com/records"
API_KEY = "free_tier_key_abc123"

MAX_PAGES = 10  # Only collect what is actually needed and permitted by TOS

records = []

for page in range(1, MAX_PAGES + 1):
    response = requests.get(API_URL, params={"page": page, "key": API_KEY})

    # Stop if the API signals there are no more pages
    data = response.json()
    if not data.get("results"):
        print(f"No more results at page {page}. Stopping.")
        break

    records.extend(data["results"])

    # Respect the API's rate limit — don't hammer the server
    time.sleep(1)

print(f"Collected {len(records)} records within allowed limits.")
```

**What changed and why:**
- `MAX_PAGES = 10` — only collect what is genuinely needed, not the maximum possible
- `if not data.get("results"): break` — stop when the API runs out of data instead of blindly looping
- `time.sleep(1)` — adds a 1-second pause between requests to respect rate limits and avoid being flagged or blocked

---

### Violation 2 — Storing Raw PII Permanently Without Anonymisation

**What is the problem?**

The comment in the original script says:
```python
# Store all records permanently in company database
save_to_database(records)
```

This stores **all raw patient records permanently**, including Direct PII fields like `full_name`, `email`, and sensitive `diagnosis_notes`. This is a serious ethical and legal violation because:

1. **Data minimisation principle (GDPR Article 5)** — you should only store what is necessary for your specific purpose. Permanent storage of raw PII is almost never justified.
2. **Purpose limitation** — the data was collected for a specific research purpose. Storing it permanently "just in case" goes beyond that purpose.
3. **Healthcare data regulations** — patient health data (diagnosis notes) is classified as **sensitive personal data** and requires explicit consent and strict handling rules.
4. **Security risk** — storing raw PII in a company database creates a data breach risk. If the database is compromised, real patient identities are exposed.

**Corrected Code:**

```python
from datetime import datetime, timedelta

def anonymise_record(record):
    """
    Removes or masks all PII fields before storing.
    Only anonymised, research-useful data is kept.
    """
    return {
        # Replace name and email with a meaningless ID
        "patient_id":       hash(record.get("email", "")),

        # Keep only birth year, not full date
        "birth_year":       record.get("date_of_birth", "")[:4],

        # Keep only first 3 digits of zip code
        "region_code":      record.get("zip_code", "")[:3],

        # Group job title into broad categories (simplified example)
        "occupation_group": "General",

        # Keep diagnosis notes but flag for manual review before use
        "diagnosis_notes":  record.get("diagnosis_notes", ""),

        # Record when this data expires — don't store forever
        "expires_at":       (datetime.now() + timedelta(days=365)).isoformat(),
    }

# Anonymise every record before it touches the database
anonymised_records = [anonymise_record(r) for r in records]

# Store only the cleaned, anonymised version — never the raw data
save_to_database(anonymised_records)

print(f"Stored {len(anonymised_records)} anonymised records (no raw PII saved).")
```

**What changed and why:**
- Raw PII fields (`full_name`, `email`) are never stored — replaced with a hashed ID
- `date_of_birth` is reduced to `birth_year` only
- `zip_code` is truncated to a regional code
- An `expires_at` field is added — data should not be stored permanently; it should be reviewed and deleted after its useful life
- The comment no longer says "permanently" — data retention must be time-limited

---

## Summary of Violations Found

| # | Violation | Risk | Fix |
|---|---|---|---|
| 1 | Fetching 100 pages without limit checks | TOS violation, excessive data collection, rate limit abuse | Cap pages, check for empty results, add `time.sleep()` |
| 2 | Storing raw PII permanently in a database | GDPR/HIPAA violation, data breach risk, purpose limitation breach | Anonymise before storing, add data expiry |
