# 🧠 Smart Job Match AI  
### Automating End-to-End Resume Parsing, Job Scraping & AI-Based Matching  

---

## 📘 Overview  
**Smart Job Match AI** is a modern AI-powered automation system designed to streamline recruitment. It automatically parses resumes, scrapes live job listings, performs AI-based job matching using **Google Gemini**, and delivers the top results to users via **email**, **Lovable dashboard**, and **Google Sheets**.  

---

## 🚀 Purpose  
Automate end-to-end resume parsing, job scraping & matching, LLM reasoning (Gemini), and result delivery with minimal manual effort.

---

## 🔍 Main Sources  
- **LinkedIn job listings** (via scraper)  
- **Google Sheets** (as a fallback job store)  
- **User resume uploads** (via Lovable Web UI)

---

## 🤖 Models  
- **Primary:** Google Gemini (through Lovable)  
- **Fallback:** Azure OpenAI  

---

## 🧩 Tools & Technologies  
- **n8n** — Workflow automation  
- **Lovable AI** — Frontend/AI agent  
- **Google Sheets** — Data storage  
- **Gmail / SMTP** — Email delivery  
- **LinkedIn Scraper** — Job ingestion (custom / Apify / Puppeteer)  
- **ngrok** — Local testing & CORS bypass  

---

## ⚙️ High-Level Flow  
**User uploads resume → Lovable UI → n8n webhook → resume extraction → job scraping (LinkedIn + Sheets) → AI-based matching → ranking → top 5 results via Gmail + Sheets + Lovable dashboard**

---

## 🧠 Workflow Snapshot  

### 🔹 Webhook Trigger (Resume Upload)
- Receives form data `{ email, resume_url, filename }`  
- Responds immediately with acknowledgment  

### 🔹 Download Resume (Optional)
- If `resume_url` is remote, downloads and stores as binary  

### 🔹 Resume Extractor  
- Parses text from PDF/DOCX or calls Gemini Vision for OCR  
- Outputs structured JSON:  
  ```json
  { "name": "", "email": "", "skills": [], "years_of_experience": "", "technologies": [] }
  ```

### 🔹 Job Sources  
- **LinkedIn Scraper** – Fetches latest jobs  
- **Google Sheets (Job Store)** – Acts as a backup  

### 🔹 Match Engine  
- Normalizes skills, experience, technologies  
- Computes fuzzy/exact overlaps and scores each job  

### 🔹 Top N Extractor  
- Slices **Top 5** job matches  

### 🔹 Gmail / Notification  
- Sends formatted HTML email with clickable job links  

### 🔹 Google Sheets Update  
- Appends or updates candidate row  

### 🔹 Lovable Webhook Response  
- Displays live job matches on the dashboard  

---

## 🧾 Example Request  
```json
{
  "email": "saikrishik989@gmail.com",
  "filename": "Krishik_Resume.pdf",
  "resume_url": "https://.../Krishik_Resume.pdf"
}
```

---

## 🧩 Key Files  
| File | Description |
|------|--------------|
| `SmartJobMatch_Automation.json` | Export of n8n workflow |
| `lovable_prompt_job_match.txt` | Lovable agent / Gemini prompt |
| `resume_extract_code.js` | Resume parsing logic |
| `match_engine_code.js` | Job matching logic |
| `top5_extractor_code.js` | Extracts top 5 matches |

---

## 💬 Lovable / Gemini Prompts  

### **Resume Extraction Prompt**
> You are an expert resume parser. Input: raw resume text or OCR. Output: strict JSON only with fields:
> ```json
> { "name":"", "email":"", "phone":"", "skills":[""], "technologies_familiar_with":[""], "years_of_experience":"" }
> ```
> Return only JSON, no extra text.

### **Job Matching Explanation Prompt**
> You are a job-match assistant. Input: resume JSON and matched jobs.  
> Output: 3–4 sentences summarizing why these 5 jobs fit best based on skills and experience.

---

## 📊 Google Sheets Config  
**Sheet A:** Job Store  
```
title | company | skills | technologies_required | years_of_experience | link | description
```
**Sheet B:** Candidates  
```
email | name | resume_filename | top_match_1_title | top_match_1_company | ...
```

---

## 🔧 Deployment & CORS  
- Use `ngrok http 5678` for local testing.  
- Configure `Access-Control-Allow-Origin` for production.  
- For Lovable integration, use server-side webhook calls.  

---

## ⚠️ Error Handling  
- **Scraper failure:** Send WhatsApp/email alert.  
- **Resume parse failure:** Ask user to re-upload.  
- **No matches found:** Return “No suitable matches found.”  

---

## ✅ Testing Checklist  
- [ ] Upload resume via Lovable UI → verify webhook trigger.  
- [ ] Confirm parsed JSON output.  
- [ ] Verify job matches in `Match Engine`.  
- [ ] Check top 5 results in Gmail + Google Sheets.  
- [ ] Ensure Lovable dashboard shows results.  

---

## 📦 Deliverables  
- ✅ Full **n8n workflow JSON** ready to import  
- ✅ **Lovable agent prompt** + response schema  
- ✅ Optional **React + Tailwind dashboard** for visualization  
- ✅ **Runbook** for deployment (ngrok → self-host → CORS → credentials)

---

## 📧 Example Output (n8n)
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

## 🧩 Author  
**👤 Sai Krishik**  
🔗 [GitHub: Krishik04](https://github.com/Krishik04)
