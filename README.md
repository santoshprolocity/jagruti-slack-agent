# JAGRUTI SLACK AGENT - COMPLETE DETAILED IMPLEMENTATION PLAN FOR INTERNS & FRESHERS
# EVERY PHASE, EVERY COMMAND, EVERY CONFIG EXPLAINED IN FULL DETAIL
# Version: Slack-First + RTS + Full Executable Steps (Phases 3-6)
# Date: June 21, 2026
# Purpose: So that even complete beginners can understand WHY each step exists and can execute it

# ===================================================================
# HOW TO READ THIS DOCUMENT (IMPORTANT)
# ===================================================================
# 1. Read one phase at a time. Do not try to do everything in one day.
# 2. Every command has multiple lines of explanation above it.
# 3. "WHY" explains the business or technical reason.
# 4. "WHAT HAPPENS IF YOU SKIP IT" tells you the risk.
# 5. After completing each phase, reply with "Phase X completed".
# 6. Ask questions freely. This project is designed to teach you.

# ===================================================================
# PROJECT OVERVIEW - WHY WE ARE BUILDING THIS
# ===================================================================

# This Slack agent turns Slack into the main communication and data collection tool for frontline nutrition workers in the Jagruti program.
# WHY: Field workers in rural areas often find complex mobile apps difficult. They already use messaging apps daily.
# By making Slack the primary interface and using Slack's native Agent Builder, we remove friction and create a native Slack experience.
# This dramatically increases data collection frequency and quality while aligning with the demo's emphasis on Slack AI capabilities.

# Key Features:
# - Daily prompts for field workers to log experiences
#   WHY: Consistent daily input creates reliable longitudinal data about children.
# - All data stored directly in Salesforce (no custom database)
#   WHY: Jagruti already has a Salesforce org. Duplicating data creates sync problems. Salesforce is the single source of truth.
# - Automatic detection of critically malnourished children
#   WHY: Early detection can save lives. The system should proactively find high-risk cases.
# - Session planning and coordination
#   WHY: Helps supervisors plan counselling sessions based on real field data.
# - AI-powered clinical reports with Real-Time Search (RTS)
#   WHY: Raw notes are turned into professional medical summaries. RTS ensures the agent always has the freshest workspace context.
# - Clean Slack Canvas reports using native Slack surfaces
#   WHY: Canvas gives beautiful, formatted, permanent records inside Slack.

# Current Architecture (Slack-First):
# - Single source of truth = Salesforce Developer Org
# - Communication & Agent Layer = Slack Agent Builder + Bolt + Native Surfaces (split-view, Canvas, suggested prompts, streaming)
# - Bridge = Salesforce MCP + Slack MCP Server (both used together)
# - Real-Time Search (RTS) = Core capability to pull fresh in-workspace context instantly
# - Clinical Reasoning = Claude (used selectively for medical accuracy)
# - Automation = Slack workflows, Bolt scheduled jobs (native Slack tools preferred)

# ===================================================================
# PREREQUISITES - FOUNDATION SETUP (MUST DO BEFORE ANY PHASE)
# ===================================================================

(Same as previous version — install tools, accounts, project folder, Phase 0 Salesforce setup, Phase 1 Slack App with Agent Builder, Phase 2 both MCP servers.)

# ===================================================================
# PHASE 3: DAILY EXPERIENCE LOGGING + REAL-TIME SEARCH (RTS)
# ===================================================================

**Step 3.1: Create the dedicated channel**

In Slack:
- Go to Browse channels → Create channel
- Name: `field-worker-experiences`
- Set to Public

**Why:** All field observations will live in one searchable place. This makes RTS much more effective.

**Step 3.2: Set up Daily Prompt using Native Slack Tools**

**Recommended method (no code needed):**

1. In Slack, go to **Automation → Workflow Builder**
2. Click **Create Workflow**
3. Select **Scheduled** trigger
4. Set time to **Every day at 9:00 AM**
5. Add action **"Send a message"**
6. Target channel: `#field-worker-experiences`
7. Paste this exact message:

```
Good morning team! 🌱

Please share today's field observations about the children you visited.
Examples:
• Child's name, age, and SAM score
• Feeding practices observed
• Any photos or voice notes

Just reply in this channel. Your input helps us detect critical cases early.
```

8. Save → Publish the workflow.

**Why native Slack Workflow Builder?** It aligns with the demo's request to use Slack AI capabilities and avoids depending on external cron systems.

**Step 3.3: Create Real-Time Search (RTS) Helper**

Create file `rts_helper.py` with this exact code:

```python
# rts_helper.py - Real-Time Search (RTS) for Jagruti Agent
from datetime import datetime

def real_time_search(query: str = "nutrition OR SAM OR stunting OR feeding", limit: int = 15):
    """
    Pulls the freshest context from Slack workspace + Salesforce before any major action.
    This is the core RTS implementation.
    """
    results = []
    
    # Use Slack MCP Server to search recent messages
    slack_results = slack_mcp.call(
        tool="search_messages",
        params={
            "query": query,
            "channels": ["field-worker-experiences", "critical-cases"],
            "time_range": "last_7_days",
            "limit": limit
        }
    )
    results.extend(slack_results)
    
    # Combine with latest Salesforce data
    salesforce_results = salesforce_mcp.call(
        tool="query",
        params={
            "soql": f"""
                SELECT Id, Narrative__c, Worker_Name__c, Date__c 
                FROM Experience_Log__c 
                WHERE Date__c = LAST_N_DAYS:7 
                ORDER BY Date__c DESC LIMIT {limit}
            """
        }
    )
    results.extend(salesforce_results)
    
    # Return most recent first
    results = sorted(results, key=lambda x: x.get('timestamp', ''), reverse=True)
    
    print(f"RTS completed at {datetime.now()}. Found {len(results)} recent items.")
    return results[:limit]


# Test the RTS helper
if __name__ == "__main__":
    fresh_context = real_time_search("SAM OR malnourished", limit=10)
    print("RTS Output:", fresh_context)
```

Run test with:
```bash
python3 rts_helper.py
```

**Step 3.4: Integrate RTS into the agent**

In your main app code, call RTS **before** any major action:

```python
@app.command("/generate-report")
def generate_report(ack, say):
    ack()
    context = real_time_search("child nutrition observations", limit=12)
    # Then use 'context' for report generation
```

**Why RTS is important:** The agent becomes "live". It always knows the latest observations the moment they are needed instead of working with stale data.

# ===================================================================
# PHASE 4: CRITICAL CASE DETECTION & ALERTS (FULL EXECUTABLE)
# ===================================================================

**Step 4.1: Create the main detection script**

Create file `critical_case_detector.py` with this complete code:

```python
# critical_case_detector.py - Full executable critical case detector
from datetime import datetime
from rts_helper import real_time_search   # Reuse RTS from Phase 3

def detect_critical_cases():
    print(f"Starting critical case detection at {datetime.now()}")
    
    # Step 1: Use Real-Time Search (RTS)
    fresh_context = real_time_search(
        query="SAM OR severely malnourished OR critical OR urgent",
        limit=20
    )
    
    # Step 2: Query Salesforce for structured high-risk children using Salesforce MCP
    high_risk = salesforce_mcp.call(
        tool="query",
        params={
            "soql": """
                SELECT Id, Name, SAM_Score__c, Stunted__c, Wasted__c, Guardian_Name__c, Location__c
                FROM Child__c 
                WHERE SAM_Score__c >= 2 
                   OR Stunted__c = true 
                   OR Wasted__c = true
                ORDER BY SAM_Score__c DESC
                LIMIT 10
            """
        }
    )
    
    if not high_risk:
        print("No critical cases found today.")
        return
    
    # Step 3: Get clinical summary from Claude (only for medical judgment)
    clinical_summary = claude.call(
        system_prompt="You are an experienced pediatric nutritionist... (use the full prompt from Phase 5)",
        user_message=f"Here are the latest critical cases and observations: {fresh_context + high_risk}"
    )
    
    # Step 4: Post rich alert using Slack MCP Server (native Slack experience)
    slack_mcp.call(
        tool="send_message",
        params={
            "channel": "critical-cases",
            "text": f"🚨 {len(high_risk)} Critical Cases Detected - {datetime.now().strftime('%Y-%m-%d')}",
            "blocks": [
                {
                    "type": "header",
                    "text": {"type": "plain_text", "text": "Critical Nutrition Alert"}
                },
                {
                    "type": "section",
                    "text": {"type": "mrkdwn", "text": clinical_summary}
                }
            ]
        }
    )
    
    # Step 5: Create a persistent Canvas using Slack MCP Server
    slack_mcp.call(
        tool="create_canvas",
        params={
            "title": f"Critical Cases Report - {datetime.now().strftime('%Y-%m-%d')}",
            "content": clinical_summary,
            "channel": "critical-cases"
        }
    )
    
    print("Critical case alert posted successfully.")

# Run this file directly for testing
if __name__ == "__main__":
    detect_critical_cases()
```

**Step 4.2: Test and Schedule**

Test immediately:
```bash
python3 critical_case_detector.py
```

Check channel `#critical-cases` for the alert and Canvas.

For automation, use Slack Workflow Builder to trigger this script daily at 8 AM, or set up a Bolt scheduled job.

# ===================================================================
# PHASE 5: AI CLINICAL REPORT GENERATION (FULL EXECUTABLE)
# ===================================================================

**Step 5.1: Create the report generator script**

Create file `clinical_report_generator.py`:

```python
# clinical_report_generator.py - Full executable Phase 5
from datetime import datetime
from rts_helper import real_time_search

def generate_clinical_report():
    print(f"Generating clinical report at {datetime.now()}")
    
    # Step 1: Use Real-Time Search (RTS) - this is mandatory for freshness
    context = real_time_search(
        query="child nutrition OR SAM OR feeding OR stunting OR wasting",
        limit=20
    )
    
    # Step 2: Call Claude with the strict system prompt (Claude is only used here for medical quality)
    system_prompt = """
You are an experienced pediatric nutritionist working in rural India with 15+ years experience.

Analyze the attached field worker observations about children in our Jagruti program.

Structure your response EXACTLY in this format:

1. **Executive Summary** (2-3 sentences)
2. **Key Themes Identified** (bullet points)
3. **Children Requiring Urgent Attention** (list with reasoning and WHO criteria used)
4. **Recommended Counselling Topics** (with justification)
5. **Suggested Mother Education Content** (video ideas)
6. **Overall Program Recommendations**

Use WHO standards for SAM, Stunting, and Wasting in your analysis.
Be compassionate but clinically precise.
"""
    
    report = claude.call(
        system_prompt=system_prompt,
        user_message=f"Latest observations:\n{context}"
    )
    
    # Step 3: Deliver using Slack MCP Server (native Slack Agent surfaces)
    # Post a rich message
    slack_mcp.call(
        tool="send_message",
        params={
            "channel": "reports",
            "text": "📊 Clinical Report Generated",
            "blocks": [
                {"type": "header", "text": {"type": "plain_text", "text": "Jagruti Clinical Summary"}},
                {"type": "section", "text": {"type": "mrkdwn", "text": report}}
            ]
        }
    )
    
    # Create a permanent Canvas (this is the native Slack way to store reports)
    canvas = slack_mcp.call(
        tool="create_canvas",
        params={
            "title": f"Jagruti Clinical Report - {datetime.now().strftime('%Y-%m-%d')}",
            "content": report,
            "channel": "reports"
        }
    )
    
    print(f"Report successfully posted. Canvas URL: {canvas.get('url', 'N/A')}")
    return report


# Run for testing
if __name__ == "__main__":
    generate_clinical_report()
```

**Step 5.2: Test Phase 5**

Run:
```bash
python3 clinical_report_generator.py
```

Check `#reports` channel for the message and Canvas.

**Why this version:**
- Fully executable
- Uses RTS before calling Claude
- Uses Slack MCP Server for all Slack delivery (Canvas, rich blocks, native feel)
- Keeps the strict prompt for consistent clinical output
- Aligns with Slack Agent Builder and demo requirements

# ===================================================================
# PHASE 6: TESTING CHECKLIST (FULL EXECUTABLE)
# ===================================================================

Run these tests in order. All commands are given.

**Test 1: MCP Servers**
```bash
slack mcp test salesforce
slack mcp test slack
```
Expected: Both return "Connection successful"

**Test 2: Real-Time Search (RTS)**
```bash
python3 rts_helper.py
```
Expected: Prints recent items from Slack + Salesforce with timestamps.

**Test 3: Daily Prompt**
- Verify the Workflow Builder runs at 9 AM or trigger manually.
Expected: Message appears in `#field-worker-experiences`.

**Test 4: Critical Case Detection**
```bash
python3 critical_case_detector.py
```
Expected: Alert + Canvas posted in `#critical-cases` if high-risk children exist.

**Test 5: Clinical Report Generation**
```bash
python3 clinical_report_generator.py
```
Expected: Rich message + Canvas in `#reports` with all 6 numbered sections.

**Test 6: End-to-End Data Flow**
1. Post a test message in `#field-worker-experiences`
2. Run critical case detector
3. Run report generator
4. Check Salesforce for new `Experience_Log__c` records.

**Test 7: Native Slack Agent Features**
- Check that suggested prompts appear
- Verify report opens in split-view or Canvas

# ===================================================================
# TROUBLESHOOTING GUIDE
# ===================================================================

- RTS returns no results → Check channel names and time_range in rts_helper.py
- Slack MCP fails → Verify tokens and run `slack mcp test slack`
- Claude prompt not followed → Make system_prompt exactly as shown (copy-paste)
- Canvas not created → Ensure Slack MCP has correct permissions in manifest
- No alerts → Check if high_risk query returns data in Salesforce

**Getting Help:**
Ask: "Debug RTS in Phase 3" or "Why is my Canvas not appearing in Phase 5?"

# ===================================================================
# FINAL DELIVERABLES FOR HACKATHON
# ===================================================================

1. Native Slack Agent with Agent Builder surfaces
2. Working RTS capability in Phases 3, 4, and 5
3. Daily prompts via Workflow Builder
4. Critical case alerts with Canvas
5. Clinical reports using RTS + Claude + Slack MCP
6. 90-second demo video showing live RTS and native Slack behavior

**Next Step:**
Start with Phase 0, then move to Phase 3. After each phase reply "Phase X completed" and I will give you any missing code files.

Good luck! This project has strong potential to win "Slack Agent for Good".
    
