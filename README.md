# Smart Job Match AI — Execution Document

## Purpose
Automate end-to-end resume parsing, job scraping & matching, LLM job reasoning (Gemini), and deliver results to the user (email + Lovable dashboard + Google Sheets).

**Main sources:** LinkedIn job listings (scraper), Google Sheets job store (fallback), user resume uploads (Lovable web UI).  
**Models:** Google Gemini (preferred) via Lovable; optional fallback to Azure OpenAI.  
**Primary tools:** n8n (workflow automation), Lovable AI (frontend/agent), Google Sheets, Gmail (or SMTP), LinkedIn scraper (custom or Apify/puppeteer), ngrok for local testing.

---

## High-level Flow (One-Line)
User uploads resume → Lovable UI posts to n8n webhook → n8n extracts resume text → run matching against job feed (LinkedIn scraper + Jobs Sheet) → score and rank → send top 5: email (Gmail), Google Sheet update, respond to Lovable webhook/dashboard.

---

## Workflow Snapshot (Nodes & Roles)

### Webhook Trigger (Webhook - Resume Upload)
- Receive form data: `{ email, resume_url, filename }` or file binary.
- Respond immediately with ack; workflow continues.
<img width="1376" height="664" alt="workflow" src="https://github.com/user-attachments/assets/77b6505f-1d73-484b-a1b0-cbd5ebf214b6" />

### Resume Extractor (Code in JavaScript - Extract Resume)
- Extract text from binary PDF/DOCX (use PDF parsing library or Gemini Vision for OCR).  
- Output structured JSON: `{ name, email, skills: [...], years_of_experience, technologies: [...] }`  
- Code parses output from Lovable/AI parsing node.

### Job Source(s)
- **LinkedIn Scraper:** Custom/Apify/puppeteer — scheduled/fetch-on-demand. Produces job objects.
- **Google Sheet:** Job Store — backup and master list of jobs to match against.

### Match Engine (Code in JavaScript - Match Jobs)
- Accepts resume JSON + job items.  
- Normalizes arrays/strings (skills, techs), computes fuzzy/exact skill overlap, tech match, experience fit.  
- Produces `matched_jobs` array sorted by score.

### Top N Extractor (Code in JavaScript - Top 5)
- Slice top 5 results ensuring fields exist (title, company, description, link, match_score).

### Email / Notification (Gmail or SMTP)
- Compose HTML email showing top 5 matches with clickable links, send to user.
- <img width="1429" height="723" alt="email " src="https://github.com/user-attachments/assets/6bd64e89-d76a-44df-8185-0b131b43eec7" />

### Google Sheets (Append or Update Row)
- Append candidate + top matches or update candidate row.

### Respond to Lovable (Respond to Webhook)
- Return JSON payload to Lovable dashboard showing top 5 matches live to user.
-<img width="1435" height="725" alt="Screenshot 4" src="https://github.com/user-attachments/assets/7ac3bcb4-06c5-4684-b769-2d5528bf7857" />
<img width="1437" height="725" alt="Screenshot 5" src="https://github.com/user-attachments/assets/12a5d0cc-a80a-4130-b9ea-1501c8f62a67" />


---

## File / Workflow Names
- **SmartJobMatch_Automation.json** — n8n workflow export  
- **lovable_prompt_job_match.txt** — prompt for Lovable agent / Gemini  
- **resume_extract_code.js** — resume extraction code node  
- **match_engine_code.js** — job matching logic  
- **top5_extractor_code.js** — top 5 extraction node  

---

## Key Node Details & Code

### Webhook Trigger (n8n)
**HTTP Method:** POST  
**Path:** `/webhook/resume`

**Example request body:**
```json
{
  "email": "saikrishik989@gmail.com",
  "filename": "Krishik_Resume.pdf",
  "resume_url": "https://.../Krishik_Resume.pdf"
}
```

### Resume Extractor (Simplified Code Node)
```javascript
let data = $json.output || $json;

if (typeof data === 'string') {
  data = data.replace(/```json|```/g,'').trim();
  try { data = JSON.parse(data); } catch(e) {}
}

const skills = data.skills || [];
const years_of_experience = data.years_of_experience || "";
const technologies = data.technologies_familiar_with || [];

return [{ json: { name: data.name||"", email: data.email||"", skills, years_of_experience, technologies } }];
```

### Match Engine (Robust Code Node)
```javascript
const resume_skill = $('Code in JavaScript1').first().json.skills;
const resume_exp = $('Code in JavaScript1').first().json.years_of_experience;
const resume_tech = $('Code in JavaScript1').first().json.technologies_familiar_with;
const jobs = $items('Get row(s) in sheet').map(i => i.json);

function normalizeArray(value){
  if(!value) return [];
  if(Array.isArray(value)) return value.map(v=>String(v).toLowerCase());
  try{ const p=JSON.parse(value); if(Array.isArray(p)) return p.map(v=>String(v).toLowerCase()); }catch(e){}
  return String(value).split(/[,;]+/).map(s=>s.trim().toLowerCase()).filter(Boolean);
}

const resumeSkills = normalizeArray(resume_skill);
const techStack = normalizeArray(resume_tech);
const exp = parseFloat(resume_exp) || 0;

let matchedJobs = [];
for(const job of jobs){
  const jobSkills = normalizeArray(job.skills);
  const jobTech  = normalizeArray(job.technologies_required || job.technologies);
  const jobExp   = parseFloat(job.years || 0);

  const skillMatchCount = resumeSkills.filter(s=>jobSkills.includes(s)).length;
  const techMatchCount  = techStack.filter(t=>jobTech.includes(t)).length;
  const score = skillMatchCount*2 + techMatchCount + (exp>=jobExp?1:0);

  if(score>0){
    matchedJobs.push({
      title: job.title || "Unknown",
      company: job.company || "Company not specified",
      description: job.description || "",
      application_link: job.link || "",
      match_score: score,
      matched_skills: skillMatchCount,
      matched_technologies: techMatchCount
    });
  }
}

matchedJobs.sort((a,b)=>b.match_score - a.match_score);
return [{ json: { matched_jobs: matchedJobs } }];
```

### Top 5 Extractor (Code Node)
```javascript
const input = $input.first().json;
const jobs = Array.isArray(input.matched_jobs) ? input.matched_jobs : [];
const top5 = jobs.slice(0,5).map(j => ({
  title: j.title, company: j.company, description: j.description,
  application_link: j.application_link, match_score: j.match_score,
  matched_skills: j.matched_skills, matched_technologies: j.matched_technologies
}));
return [{ json: { email: input.email || "", top_matches: top5 } }];
```

### Gmail Node (HTML Email)
```html
<h2>Top 5 Job Matches</h2>
<ul>
{{ ($json.top_matches || []).map(job => `
  <li>
    <strong>${job.title}</strong> — ${job.company}<br/>
    Score: ${job.match_score}<br/>
    ${job.description ? job.description + '<br/>' : ''}
    <a href="${job.application_link || '#'}">Apply</a>
  </li>
`).join('') }}
</ul>
```

---

## Lovable / Gemini Prompts

### Resume → Extract Fields Prompt
```
You are an expert resume parser. Input: raw resume text or OCR.
Output strict JSON only:
{ "name":"", "email":"", "phone":"", "skills":["..."], "technologies_familiar_with":["..."], "years_of_experience":"" }
```

### Job Matching Explanation Prompt
```
You are a job-match assistant. Input: resume JSON + top matched jobs.
Produce 3–4 sentences explaining why these jobs fit best based on skills & experience.
```

---

## Google Sheets Config
**Sheet A:** Job Store — `title, company, skills, technologies_required, years of experience, link, description`  
**Sheet B:** Candidates — `email, name, resume_filename, top_match_1_title, top_match_1_company, ...`

---

## LinkedIn Scraper Guidance
If using Apify or custom puppeteer:
- Respect robots.txt & rate limits.
- Save fields: id, link, title, company, location, description, employmentType, postedAt, applyUrl.
- Structure must match jobs used in Match Engine.

---

## Deployment & CORS
- For local testing: `ngrok http 5678` and use public URL in Lovable or frontend.
- In production: host n8n on HTTPS and configure `Access-Control-Allow-Origin`.

---

## Error Handling
- **Scraper failure:** notify admin via WhatsApp/email.  
- **Resume parse failure:** prompt user to re-upload.  
- **No matches:** “No suitable matches found. Try updating skills or expand job filters.”

---

## Testing Checklist
✅ Upload a resume via Lovable UI → confirm webhook execution in n8n.  
✅ Resume Extractor outputs valid JSON.  
✅ Match Engine consumes JSON and job list correctly.  
✅ Top 5 node returns exactly 5 matches.  
✅ Gmail sends formatted email.  
✅ Google Sheets append/update verified.  
✅ Lovable displays top 5 jobs live.

---

## Example Payloads

### Webhook Input
```json
{
  "email": "saikrishik989@gmail.com",
  "filename": "Krishik_Resume.pdf",
  "resume_url": "https://.../Krishik_Resume.pdf"
}
```

### Typical Output
```json
{
  "matched_jobs": [
    {
      "title": "Data Scientist",
      "company": "Clear Ridge Defense",
      "match_score": 26,
      "matched_skills": 13,
      "matched_technologies": 0,
      "application_link": "https://..."
    }
  ]
}
```

---

## Deliverables
- ✅ Full n8n workflow export — **SmartJobMatch_Automation.json**
- ✅ Lovable agent prompt + webhook schema
- ✅ React + Tailwind dashboard prompt (optional)
- ✅ Deployment guide (ngrok → self-host → CORS → credentials)

---
