# Active Shielding with Model Armor

Two paste blocks create the two templates (Tasks 2 and 3). Tasks 1, 4 and 5 are console steps, follow the lab for those. Every block ends with a check line, compare it with the note under the block.

## Known lab traps/bugs (read first)

1. Broken lab text in Task 3: the two template paths are printed as projects//locations/... with the project ID missing. Pasted like that, the template is wrong. The correct form is projects/YOUR_PROJECT_ID/locations/us-central1/... (the blocks here fill it in for you).
2. The gcloud model-armor commands are denied for student accounts (PERMISSION_DENIED: Read access to project was denied). This tutorial talks to Model Armor with curl instead.
3. Task 4 only works after both templates exist, and the notebook must be run top to bottom. A skipped cell means missing log entries and a failed check in Task 4 or Task 5.
4. Logs and the checker lag. After the notebook finishes wait 1-2 minutes, and if a check fails wait 30-60 seconds and click again.

## Setup

Nothing to edit. The lab fixes the region (us-central1) and all four names.

```bash
gcloud config get-value project > ~/project
echo "us-central1" > ~/region
echo "projects/`cat ~/project`/locations/`cat ~/region`" > ~/loc
echo "https://modelarmor.`cat ~/region`.rep.googleapis.com/v1/`cat ~/loc`/templates" > ~/url
curl -s -H "Authorization: Bearer `gcloud auth print-access-token`" \
  "https://dlp.googleapis.com/v2/`cat ~/loc`/inspectTemplates/advisor-inspect-template" | grep '"name"'
curl -s -H "Authorization: Bearer `gcloud auth print-access-token`" \
  "https://dlp.googleapis.com/v2/`cat ~/loc`/deidentifyTemplates/advisor-deidentify-template" | grep '"name"'
cat ~/loc ~/url
```

You should see name lines for advisor-inspect-template and advisor-deidentify-template (the lab made them for you), then your project path and the Model Armor address. If the two name lines are missing, wait a minute and paste again: the lab is still provisioning.

## Task 1. Explore the Model Armor dashboard

Follow the lab: Security > Security Command Center > Model Armor, and look at Floor Settings and Templates. No checkpoint.

## Task 2. Create the Input Sanitization Template

What the lab's form asks, as one request: malicious links on, prompt injection at medium and above, basic sensitive data detection, all four Responsible AI categories at High, both logs on, inspect and block.

```bash
cat > ~/input.json <<'EOF'
{
  "filterConfig": {
    "raiSettings": {
      "raiFilters": [
        {"filterType": "HATE_SPEECH", "confidenceLevel": "HIGH"},
        {"filterType": "DANGEROUS", "confidenceLevel": "HIGH"},
        {"filterType": "SEXUALLY_EXPLICIT", "confidenceLevel": "HIGH"},
        {"filterType": "HARASSMENT", "confidenceLevel": "HIGH"}
      ]
    },
    "piAndJailbreakFilterSettings": {"filterEnforcement": "ENABLED", "confidenceLevel": "MEDIUM_AND_ABOVE"},
    "maliciousUriFilterSettings": {"filterEnforcement": "ENABLED"},
    "sdpSettings": {"basicConfig": {"filterEnforcement": "ENABLED"}}
  },
  "templateMetadata": {
    "logTemplateOperations": true,
    "logSanitizeOperations": true,
    "enforcementType": "INSPECT_AND_BLOCK"
  }
}
EOF
curl -s -X POST -H "Authorization: Bearer `gcloud auth print-access-token`" \
  -H "Content-Type: application/json" -d @"`echo ~`/input.json" \
  "`cat ~/url`?template_id=advisor-input-filter"
curl -s -H "Authorization: Bearer `gcloud auth print-access-token`" \
  "`cat ~/url`/advisor-input-filter" | grep '"name"'
```

You should see the template echoed back with no "error", then one name line ending in advisor-input-filter.

Check my progress: Create the Input Sanitization Template

## Task 3. Create the Response Sanitization Template

Prompt injection off (that attack comes in through prompts, not answers), advanced sensitive data detection using the lab's two ready-made templates, all four Responsible AI categories at High, both logs on, inspect and block.

```bash
cat > ~/output.json <<'EOF'
{
  "filterConfig": {
    "raiSettings": {
      "raiFilters": [
        {"filterType": "HATE_SPEECH", "confidenceLevel": "HIGH"},
        {"filterType": "DANGEROUS", "confidenceLevel": "HIGH"},
        {"filterType": "SEXUALLY_EXPLICIT", "confidenceLevel": "HIGH"},
        {"filterType": "HARASSMENT", "confidenceLevel": "HIGH"}
      ]
    },
    "piAndJailbreakFilterSettings": {"filterEnforcement": "DISABLED"},
    "sdpSettings": {"advancedConfig": {
      "inspectTemplate": "LOC_X/inspectTemplates/advisor-inspect-template",
      "deidentifyTemplate": "LOC_X/deidentifyTemplates/advisor-deidentify-template"
    }}
  },
  "templateMetadata": {
    "logTemplateOperations": true,
    "logSanitizeOperations": true,
    "enforcementType": "INSPECT_AND_BLOCK"
  }
}
EOF
sed -i "s|LOC_X|`cat ~/loc`|g" ~/output.json
curl -s -X POST -H "Authorization: Bearer `gcloud auth print-access-token`" \
  -H "Content-Type: application/json" -d @"`echo ~`/output.json" \
  "`cat ~/url`?template_id=advisor-output-filter"
curl -s -H "Authorization: Bearer `gcloud auth print-access-token`" \
  "`cat ~/url`/advisor-output-filter" | grep -E '"name"|Template"'
```

You should see the template echoed back with no "error", then a name line ending in advisor-output-filter and the inspectTemplate and deidentifyTemplate lines with your project ID in them (not projects//).

Check my progress: Create the Response Sanitization Template

## Task 4. Test Defenses in Agent Platform Workbench

Follow the lab's steps, they are short: Agent Platform > Notebooks > Workbench > cymbal-testing-instance > Open JupyterLab > model_armor_test.ipynb > kernel Python 3. Then run every cell from top to bottom with SHIFT+ENTER and wait for each output before the next.

What to notice while it runs: Test 1 (jailbreak) and Test 2 (bad link) come back MATCH_FOUND from advisor-input-filter; Test 3 comes back from advisor-output-filter with the personal data swapped for tags such as [CREDIT_CARD_NUMBER].

Check my progress: Test Defenses in Agent Platform Workbench

## Task 5. Audit Flagged Violations in Cloud Logging

Wait a minute after the notebook, then confirm from Cloud Shell that the log entries arrived:

```bash
gcloud logging read 'resource.type:"modelarmor.googleapis.com"' --freshness=30m --limit=30 --format=json | grep -c 'SanitizeOperationLogEntry'
```

You should see a number above 0. If it is 0, wait a minute and run it again; if it stays 0, a notebook cell did not run.

Then do the lab's console steps: Logging > Logs Explorer, paste the lab's filter with its copy button, Run query, expand an entry and read operationType, sanitizationInput.text, filterMatchState, filterResults and sanitizationVerdict.

Check my progress: Audit Flagged Violations in Cloud Logging

## Quiz

This lab has no quiz questions.
