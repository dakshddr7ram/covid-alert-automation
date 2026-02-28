# 🛡️ Automated COVID-19 Crisis Intelligence Pipeline

## 1. Executive Summary

### The Business Problem
Government health agencies often struggle with "Data Overload"—receiving raw spreadsheets or numbers without actionable context. Critical trends are often missed until it is too late.

### The Solution
An automated pipeline that:
- Ingests daily COVID-19 data from a Postgres data warehouse
- Analyzes trends using SQL Window Functions to detect specific risk patterns (e.g., "Vaccine Stall")
- Synthesizes a strategic intelligence briefing using Google Gemini (GenAI)
- Delivers a formatted HTML report to stakeholders via Email

### Tech Stack
| Component | Technology |
|-----------|-----------|
| **Orchestration** | n8n (Workflow Automation) |
| **Database** | PostgreSQL (Data Warehousing) |
| **Intelligence** | Google Gemma 3 (27b) |
| **Delivery** | SMTP / HTML Email |

---

## 2. Architecture Overview

The pipeline moves from **Raw Data → Actionable Intelligence** in four distinct stages:

1. **Risk Detection (SQL Layer)**: Filters rows down to critical "At Risk" states using mathematical logic
2. **Aggregation (Code Layer)**: Transforms individual row data into a consolidated dataset for the AI
3. **Reasoning (AI Layer)**: A Large Language Model (LLM) interpreting the numbers and generating strategic advice
4. **Distribution (Notification Layer)**: Sends a clean, professional HTML report to decision-makers

---

## 3. Workflow Configuration (Step-by-Step)

![n8n COVID Alert Workflow](https://github.com/user-attachments/assets/a9c5863b-d1cb-4387-a79d-d3cc6dc2451a)

### 🟢 Node 1: PostgreSQL (The Filter)

**Purpose**: Extract only the states that require attention. We do the heavy calculation inside the database.

**Key Logic**:
- Calculates `daily_vax_growth_pct` using LAG() window functions
- Applies a filter: (Low Vax Growth) AND (Rising Cases) AND (Cases > 500)

**SQL Query**:
```sql
WITH history AS (
    SELECT 
        state,
        date,
        daily_new_cases,
        positive_test_rate,
        population,
        vaccination_rate,
        
        LAG(daily_new_cases, 1) OVER (PARTITION BY state ORDER BY date) as cases_1_day_ago,
        LAG(daily_new_cases, 2) OVER (PARTITION BY state ORDER BY date) as cases_2_days_ago,
        LAG(daily_new_cases, 3) OVER (PARTITION BY state ORDER BY date) as cases_3_days_ago,
        
        vaccination_rate - LAG(vaccination_rate, 1) OVER (PARTITION BY state ORDER BY date) AS daily_vax_growth_pct

    FROM covid_summary_clean
    WHERE date >= '2021-07-01' 
),
risk_analysis AS (
    SELECT 
        *,
        -- RISK A: 3 Consecutive Days of Rising Cases
        CASE 
            WHEN daily_new_cases > cases_1_day_ago 
             AND cases_1_day_ago > cases_2_days_ago 
             AND cases_2_days_ago > cases_3_days_ago 
            THEN TRUE ELSE FALSE 
        END as risk_consecutive_rise,

        -- RISK B: High Positivity Rate (> 10%)
        CASE 
            WHEN positive_test_rate > 10 THEN TRUE ELSE FALSE 
        END as risk_high_positivity,

        -- RISK C: Vaccination Stalled & High Volume Cases (> 500)
        CASE 
            WHEN daily_vax_growth_pct < 0.1 
             AND daily_new_cases > cases_1_day_ago 
             AND daily_new_cases > 500
            THEN TRUE ELSE FALSE 
        END as risk_vax_stall
    FROM history
)
SELECT 
    state,
    date,
    daily_new_cases,
    positive_test_rate,       
    population,              
    vaccination_rate,        
    CONCAT_WS(' | ', 
        CASE WHEN risk_consecutive_rise THEN '⚠️ Outbreak Trend: Cases rising for 3 consecutive days' END,
        CASE WHEN risk_high_positivity THEN '⚠️ High Positivity: Rate > 10%' END,
        CASE WHEN risk_vax_stall THEN '⚠️ Vulnerability: Vaccination stalled while cases are rising' END
    ) as risk_summary
FROM risk_analysis
WHERE 
    date = '2021-08-11'
    AND (risk_consecutive_rise OR risk_high_positivity OR risk_vax_stall);
```

---

### 🟠 Node 2: Code Node (The Aggregator)

**Purpose**: Batch processing. Instead of sending 5 separate emails (which spams the user and hits API limits), we combine all risky states into one context object.

**Language**: JavaScript

**Key Code**:
```javascript
const items = $input.all();
if (items.length === 0) {
  return [{ json: { full_report: "No critical risks detected today." } }];
}
let reportText = "";
items.forEach((item, index) => {
  const s = item.json;
  const casesPerMillion = Math.round((s.daily_new_cases / s.population) * 1000000);
  reportText += `### ${index + 1}. ${s.state} (Pop: ${(s.population / 1000000).toFixed(1)}M)\n`;
  reportText += `   - 🦠 New Cases: ${s.daily_new_cases} (${casesPerMillion} per million)\n`;
  reportText += `   - 🧪 Positivity: ${s.positive_test_rate}%\n`;
  reportText += `   - 💉 Vax Rate: ${s.vaccination_rate}%\n`;
  reportText += `   - ⚠️ Flag: ${s.risk_summary}\n\n`;
});
return [{
  json: {
    report_date: items[0].json.date,
    total_states: items.length,
    full_report: reportText
  }
}];
```

---

### 🔵 Node 3: AI Reasoning

**Purpose**: Turns raw numbers into human-readable strategy.

**Model**: Gemma 3 (27b)

**Prompt Strategy**: 
- "Role Prompting" (Acting as Chief Data Officer)
- "Chain of Thought" (Analyze → Conclude)

**Key Instruction**: "Generate the report in HTML format directly to ensure perfect rendering."

---

### 🟣 Node 4: Email Delivery

**Purpose**: Sends the final report.

**Configuration**:
- **Subject**: ⚠️ CRITICAL COVID-19 ALERT
- **Format**: HTML
- **Cleanup**: Uses `{{ $json.text.replace(/\`\`\`html/g, '').replace(/\`\`/g, '') }}` to remove Markdown wrappers that might break the email rendering

---

## Getting Started

1. Set up a PostgreSQL database with COVID-19 data
2. Configure n8n workflow with the provided nodes
3. Add your email SMTP credentials
4. Configure Google Gemini API access
5. Deploy and schedule the workflow

## License

This project is open source and available under the MIT License.
