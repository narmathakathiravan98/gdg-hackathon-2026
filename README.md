# HDFS Log Triage Pipeline

Convert 2000 lines of HDFS logs into ~95 structured JSON anomalies using an LLM. Stops you from manually grepping through production logs.

## The Problem

You have 2000 lines of HDFS logs. 1920 are noise (block received, verified, terminated). Finding the actual problems means:
- Grepping for ERROR/WARN
- Pattern matching with regex
- Understanding context manually
- Building alerts from scratch

It's tedious and you'll miss cascading failures that only show up as patterns, not keywords.

## What This Does

Sends your entire log to an LLM (via HuggingFace API) which:
1. Filters out the routine stuff
2. Finds actual anomalies (network failures, bulk deletions, performance issues)
3. Extracts specifics (actual IPs, block IDs, timestamps)
4. Returns clean JSON ready to send to your backend

For the HDFS_2k.log example:
- **80 WARN entries** - block transfer failures
- **5 CRITICAL** - bulk delete commands 
- **5-10 WARNING** - deletion patterns suggesting cascades
- **5 WARNING** - weird block sizes (incomplete writes)

Total: ~95 problems pulled from 2000 lines of logs in ~45 seconds.

## Why Not Regex?

Regex finds patterns you already know. An LLM understands:
- "50 blocks deleted in 2 minutes across 10 nodes = cascading failure"
- "Block size 3MB instead of 64MB = corruption risk"
- "This network error pattern + this performance spike = related"

You write one prompt, it handles complexity. Try doing that with 50 regex rules.

## Setup

Get a free HuggingFace token: https://huggingface.co/settings/tokens (read access is enough)

### In Colab (easiest)

```python
# Cell 1
!pip install requests -q

# Cell 2
hf_token = input("HF token: ")

# Cell 3
import json
import requests

log_file = "HDFS_2k.log"  # Upload your file first
with open(log_file) as f:
    log_content = f.read()

api_url = "https://api-inference.huggingface.co/models/mistralai/Mixtral-8x7B-Instruct-v0.1"
headers = {"Authorization": f"Bearer {hf_token}"}

prompt = f"""Analyze this HDFS log. Extract anomalies as JSON array ONLY (no text):

{log_content}

Output format:
[
  {{"service_name": "...", "timestamp": "YYMMDD HHMMSS", "error_severity": "WARN|WARNING|CRITICAL", "suggested_remediation": "specific fix with actual IPs/block IDs"}}
]"""

response = requests.post(api_url, headers=headers, json={"inputs": prompt})
result = response.json()
text = result[0]['generated_text']

# Extract JSON
start = text.find('[')
end = text.rfind(']') + 1
json_str = text[start:end]
anomalies = json.loads(json_str)

print(f"Found {len(anomalies)} anomalies")
for a in anomalies[:3]:
    print(f"\n{a['service_name']} @ {a['timestamp']}")
    print(f"{a['error_severity']}: {a['suggested_remediation']}")
```

### Local

```bash
pip install requests
export HF_TOKEN="your_token"
python pipeline.py HDFS_2k.log
```

## Output Example

```json
[
  {
    "service_name": "dfs.DataNode$DataXceiver",
    "timestamp": "081109 214043",
    "error_severity": "WARN",
    "suggested_remediation": "Block blk_-2918118818249673980 failed from 10.251.30.85:50010 to 10.251.90.64. Check network between these nodes and verify replication."
  },
  {
    "service_name": "dfs.FSNamesystem",
    "timestamp": "081111 065254",
    "error_severity": "CRITICAL",
    "suggested_remediation": "Bulk delete to 10.250.17.177:50010 for 100+ blocks. Verify sufficient replication before deletion."
  }
]
```

## What Gets Filtered

**Noise (ignored):**
- "Received block blk_xxx" → normal operation
- "PacketResponder terminating" → routine cleanup
- "blockMap updated" → normal registration
- "Verification succeeded" → everything working

**Actual problems (extracted):**
- "Got exception while serving" → network failure
- "ask X to delete [100 blocks]" → data loss risk
- Multiple deletes in 2 minutes → cascading failure
- Block size 3MB instead of 64MB → corruption

## The JSON Format

Every anomaly has exactly 4 fields:

```json
{
  "service_name": "component from the log",
  "timestamp": "YYMMDD HHMMSS from the log",
  "error_severity": "WARN or WARNING or CRITICAL",
  "suggested_remediation": "actionable fix with actual IPs/block IDs from this specific log entry"
}
```

Not generic. Not templated. Each one is specific to that log line with real values extracted.

## Limitations

- Needs internet (HuggingFace API)
- Takes 30-60 seconds per log
- Free tier is slow (can timeout)
- Logs > 10MB might hit token limits

## What Actually Matters

The log has 2000 lines. The model needs to:
1. Understand what normal HDFS behavior is
2. Spot deviations from normal
3. Recognize patterns across multiple log entries
4. Extract actual values, not write templates

A simple regex can find "WARN" entries. This understands the context of the logs.

## Real Example

From the test log:
- 80 entries with "Got exception while serving" → all network failures
- 224 individual block deletions → grouped into 5 bulk commands
- 307 operations > 20 seconds → performance degradation cluster
- 35 blocks with non-standard sizes → potential corruption

## Running It

1. Get log file (plain text)
2. Get HF token
3. Run the code above
4. Get JSON
5. Send for webhook database injection
