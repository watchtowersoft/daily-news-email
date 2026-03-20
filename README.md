# daily-news-email

AWS-driven daily email that delivers a balanced, AI-curated morning news digest to your inbox every day.

## Overview

A Python script that:
1. Searches the web for today's top news across 5 categories using Tavily
2. Feeds the results to Claude (Anthropic) to write a clean HTML email digest
3. Sends the email via AWS SES
4. Runs automatically every morning via AWS Lambda + EventBridge

## News Sections

1. **Technology & AI** (AI developments, major tech companies, emerging trends)
2. **World / Geopolitics** (international news, conflict, diplomacy)
3. **Business & Markets** (stocks, economy, corporate news)
4. **Science & Health** (research, medicine)
5. **US Politics** (government, congress, policy)

## Project Structure

```
daily-news-email/
├── main.py              # Run locally for testing
├── lambda_function.py   # Deployed to AWS Lambda
├── requirements.txt     # Python dependencies
├── output.html          # Last generated email preview (local only, not committed)
└── .env                 # Local secrets (never commit this)
```

## Prerequisites

- Python 3.11+
- Anthropic API account — console.anthropic.com (paid, ~$0.10–0.30/run)
- Tavily API account — tavily.com (free tier: 1,000 searches/month)
- AWS account with SES and Lambda access

## Local Setup

**1. Clone and create a virtual environment:**
```bash
python3 -m venv venv
source venv/bin/activate  # Mac/Linux
pip install -r requirements.txt
```

**2. Create a `.env` file in the project root:**
```
ANTHROPIC_API_KEY=sk-ant-...
TAVILY_API_KEY=tvly-...
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_REGION=us-east-1
EMAIL_FROM=you@gmail.com
EMAIL_TO=you@gmail.com
```

**3. Run locally:**
```bash
python main.py
```
This saves a preview to `output.html` and sends the email.

## AWS Setup

### SES — Verify your email address
1. Go to **AWS Console → SES → Verified identities**
2. Click **Create identity** → choose **Email address**
   > Important: use the **Verified identities** page directly, not the "Get set up" wizard which forces domain setup
3. Enter your sender email and click **Create identity**
4. Click the verification link in the email AWS sends you
5. Repeat for the recipient email if it's different

> **SES Sandbox:** By default SES only allows sending to verified addresses. To send to anyone, request production access via **Account dashboard → Request production access**.

### IAM — Create credentials for local runs
1. Go to **AWS Console → IAM → Users → Create user**
2. Name it `daily-news-email`
3. Attach policy: `AmazonSESFullAccess`
4. Open the user → **Security credentials → Create access key**
5. Choose **"Application running outside AWS"**
6. Copy the Access key ID and Secret access key into your `.env` file

### Lambda — Deploy the function

**1. Build the deployment package for Linux:**

> This step is critical — if you build on a Mac without targeting Linux, you'll get a `pydantic_core` import error in Lambda.

```bash
rm -rf package lambda_package.zip
mkdir package
pip install anthropic requests \
  --platform manylinux2014_x86_64 \
  --target package \
  --only-binary=:all: \
  --python-version 3.12
cp lambda_function.py package/
cd package && zip -r ../lambda_package.zip . && cd ..
```

**2. Create the Lambda function:**
- Go to **AWS Console → Lambda → Create function**
- Choose **Author from scratch**
- Name: `daily-news-email`
- Runtime: **Python 3.12**
- Click **Create function**

**3. Upload the zip:**
- In the Lambda function page → **Code → Upload from → .zip file**
- Upload `lambda_package.zip`

**4. Set environment variables:**
- Go to **Configuration → Environment variables → Edit → Add environment variable**
- Add these (do NOT add `AWS_REGION` — it's reserved and set automatically by Lambda):
  ```
  ANTHROPIC_API_KEY    → your sk-ant-... key
  TAVILY_API_KEY       → your tvly-... key
  EMAIL_FROM           → your verified sender email
  EMAIL_TO             → your verified recipient email
  ```

**5. Give Lambda permission to send via SES:**
- Go to **Configuration → Permissions → click the execution role name**
- In IAM → **Add permissions → Attach policies**
- Add `AmazonSESFullAccess`

**6. Increase the timeout:**
- Go to **Configuration → General configuration → Edit**
- Set timeout to **3 min 0 sec** (default 3 seconds is too short)
- Click **Save**

**7. Test it:**
- Click **Test** in the Lambda console
- Create a test event with an empty JSON body `{}`
- Click **Test** and check the output — you should see `"statusCode": 200`

### EventBridge — Schedule daily delivery
1. Go to **AWS Console → EventBridge → Rules → Create rule**
2. Name it `daily-news-email-trigger`
3. Rule type: **Schedule** → click **Next**
4. Schedule pattern: **Cron-based** — enter your desired time
   - EventBridge lets you set a timezone directly — no UTC conversion needed
   - Example: 7:00 AM, Eastern Time
5. Click **Next** → Target: **Lambda function** → select `daily-news-email`
6. Click through to **Create rule**

> EventBridge automatically adds the permission for itself to invoke your Lambda function — no manual IAM changes needed.

## Email Format

- **Subject:** 🗞️ Your Morning Briefing — [Date]
- **Header:** Daily Morning Briefing with today's date
- **Sections** with color-coded left border accents:
  - Technology & AI — blue
  - World / Geopolitics — green
  - Business & Markets — orange
  - Science & Health — teal
  - US Politics — purple
- **Footer:** Source attribution and verification reminder

## Quality Standards

- Factual and neutral — no political framing, loaded language, or editorializing
- Stories from the past 24 hours only
- Duplicate stories synthesized into one entry
- ⚖️ "Perspectives vary" flag added when outlets have conflicting coverage
- "No major developments today" used when a section is genuinely slow

## Cost Estimate (per run)

| Service | Cost |
|---|---|
| Anthropic (Claude Opus) | ~$0.10–0.30 |
| Tavily (13 searches) | Free tier |
| AWS Lambda | Free tier |
| AWS SES | ~$0.0001 |
| **Total** | **~$0.10–0.30/day** |
