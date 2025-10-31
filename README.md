Purpose

Automate end-to-end resume parsing, job scraping & matching, LLM job reasoning (Gemini), and deliver results to the user (Email + Lovable Dashboard + Google Sheets).

Main Sources

LinkedIn job listings (scraper)

Google Sheets Job Store (fallback)

User Resume Uploads (Lovable Web UI)

Models

Google Gemini (preferred) via Lovable

Azure OpenAI (optional fallback)

Primary Tools

n8n – Workflow automation

Lovable AI – Frontend/agent

Google Sheets – Job storage

Gmail (or SMTP) – Email delivery

LinkedIn Scraper – Custom or Apify/Puppeteer

ngrok – Local testing and webhook tunneling

⚙️ High-Level Flow (One-Line)
User uploads resume → Lovable UI posts to n8n webhook → n8n extracts resume text → 
run matching against job feed (LinkedIn scraper + Jobs Sheet) → score and rank → 
send top 5: Email (Gmail), Google Sheet update, respond to Lovable webhook/dashboard.

🧩 Workflow Snapshot (Nodes & Roles)
1. Webhook Trigger (Webhook - Resume Upload)

Receives form data:

{ "email": "saikrishik989@gmail.com", "resume_url": "https://.../Krishik_Resume.pdf", "filename": "Krishik_Resume.pdf" }


Responds immediately with ACK; workflow continues.

2. (Optional) Download Resume

If resume_url is remote/blob, download file; ensure binary exists.

3. Resume Extractor (Code Node – JavaScript)

Extract text from binary PDF/DOCX or use Gemini AI for OCR.

Outputs structured JSON:

{ "name": "", "email": "", "skills": [], "years_of_experience": "", "technologies": [] }


Example Code:

let data = $json.output || $json;
if (typeof data === 'string') {
  data = data.replace(/```json|```/g,'').trim();
  try { data = JSON.parse(data); } catch(e) {}
}
const skills = data.skills || [];
const years_of_experience = data.years_of_experience || "";
const technologies = data.technologies_familiar_with || [];
return [{ json: { name: data.name||"", email: data.email||"", skills, years_of_experience, technologies } }];


💡 If pdf-parse isn’t available, upload to a small serverless function or use Gemini Vision for OCR.

4. Job Sources

LinkedIn Scraper: Custom/Apify/Puppeteer (scheduled or on-demand).

Google Sheet (Job Store): Backup and master list of jobs.

5. Match Engine (Code Node – JavaScript)

Matches parsed resume against job data.

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
  const jobSkills = normalizeArray(job.skills || job['skills']);
  const jobTech  = normalizeArray(job['technologies required'] || job.technologies_required || job.technologies);
  const jobExp   = parseFloat(job['years of experience'] || job.years || 0);

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

6. Top 5 Extractor (Code Node)
const input = $input.first().json;
const jobs = Array.isArray(input.matched_jobs) ? input.matched_jobs : [];
const top5 = jobs.slice(0,5).map(j => ({
  title: j.title, company: j.company, description: j.description,
  application_link: j.application_link, match_score: j.match_score,
  matched_skills: j.matched_skills, matched_technologies: j.matched_technologies
}));
return [{ json: { email: input.email || $node["Webhook - Resume Upload"].json.body.email || "", top_matches: top5 } }];

7. Gmail / Notification Node

Use HTML email template:

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

8. Respond to Lovable (Respond to Webhook Node)
{
  "success": true,
  "message": "Top 5 job matches",
  "emailSentTo": "{{ $json.email }}",
  "top_5_jobs": [
    { "title": "", "match_score": 0, "company": "", "application_link": "" }
  ]
}

9. (Optional) Logging & Monitoring

Slack/Discord notifications for failures

Logs stored in DB

📁 File & Workflow Names
File Name	Description
SmartJobMatch_Automation.json	n8n workflow export
lovable_prompt_job_match.txt	Lovable / Gemini agent prompt
resume_extract_code.js	Resume extraction node
match_engine_code.js	Job matching logic
top5_extractor_code.js	Extracts top 5 job matches
💬 Lovable / Gemini Prompts
Resume Extraction Prompt

You are an expert resume parser. Input: raw resume text or OCR.
Output strict JSON only:

{ "name":"", "email":"", "phone":"", "skills":[""], "technologies_familiar_with":[""], "years_of_experience":"" }

Job Matching Explanation Prompt

You are a job-match assistant. Input: resume JSON + matched job objects.
Produce a 3–4 sentence summary explaining why the top 5 jobs were chosen based on skills and experience.

📊 Google Sheets Configuration
Sheet A: Job Store

| title | company | skills | technologies_required | years of experience | link | description |

Sheet B: Candidates

| email | name | resume_filename | top_match_1_title | top_match_1_company | ... |

Append or update based on email column.

🕸 LinkedIn Scraper Guidance

Use Apify or Puppeteer.

Respect robots.txt and rate limits.

Fields: id, link, title, company, location, description, employmentType, postedAt, applyUrl.

🌐 Deployment & CORS

Use ngrok http 5678 for local webhook testing.

In production, host n8n on HTTPS and configure Access-Control-Allow-Origin.

Optionally call n8n server-side via Lovable to bypass CORS.

⚠️ Error Handling

Scraper failure: Send WhatsApp/Email alert.

Resume parse failure: Ask user to re-upload.

No matches: Return: “No suitable matches found. Try updating skills or expand job filters.”

✅ Testing Checklist

Upload a sample resume via Lovable UI → Confirm n8n webhook receives data.

Verify Resume Extractor outputs parsed JSON.

Confirm Match Engine consumes resume + jobs list correctly.

Check Top5 Extractor returns 5 valid results.

Validate Gmail Node sends formatted email.

Confirm Google Sheets logs the candidate.

Verify Lovable displays top 5 jobs in the dashboard.

🧾 Example Payloads

Webhook Input

{
  "email": "saikrishik989@gmail.com",
  "filename": "Krishik_Resume.pdf",
  "resume_url": "https://.../Krishik_Resume.pdf"
}


Matched Jobs Output

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
