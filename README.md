# 🤖 Resilient Web Automation Framework (AI Browser + Captcha Solver)

## Project Description

This repository provides a robust, high-uptime framework for web automation, combining advanced AI-driven browser simulation (using Playwright/Puppeteer) with a dedicated CAPTCHA resolution service ([CapSolver](https://dashboard.capsolver.com/dashboard/overview/?utm_source=github&utm_medium=article&utm_campaign=ai-browser-captcha-solver)). This synergy ensures your automation pipelines maintain 99%+ stability against modern anti-bot systems like reCAPTCHA and Cloudflare.

## ✨ Features

*   **99%+ Uptime:** Achieved by programmatically handling CAPTCHA challenges as a failover mechanism.
*   **AI Browser Stealth:** Utilizes techniques like fingerprint spoofing and behavioral jitter to minimize initial detection.
*   **Multi-Challenge Support:** Built-in logic for reCAPTCHA v2, reCAPTCHA v3 (score boosting), and Cloudflare challenges.
*   **Asynchronous Resolution:** Offloads CAPTCHA solving to a high-speed external API, preventing blocking of the main automation thread.
*   **Modular Python Implementation:** Easy-to-integrate utility function for API interaction.

## 🛠️ Prerequisites

*   Python 3.8+
*   `requests` library (`pip install requests`)
*   An AI Browser library (e.g., Playwright or Puppeteer)
*   A [CapSolver API Key](https://dashboard.capsolver.com/dashboard/overview/?utm_source=github)

## 🚀 Quick Start

### 1. Installation

\`\`\`bash
# Install required Python packages
pip install requests
# Install your preferred browser automation library (e.g., Playwright)
pip install playwright
playwright install
\`\`\`

### 2. Configuration

Set your CapSolver API key as an environment variable:

\`\`\`bash
export CAPSOLVER_API_KEY="YOUR_API_KEY_HERE"
\`\`\`

### 3. Core Solver Utility (`solver_utils.py`)

Use the following function to interact with the CapSolver API. This is the core component for achieving stability.

\`\`\`python
import requests
import time
import os

# CapSolver API endpoints
API_URL = "https://api.capsolver.com/createTask"
GET_RESULT_URL = "https://api.capsolver.com/getTaskResult"
CLIENT_KEY = os.getenv("CAPSOLVER_API_KEY")

def solve_recaptcha_v2(site_key: str, page_url: str) -> str | None:
    """
    Submits a reCAPTCHA v2 task to CapSolver and retrieves the solution token.
    
    Args:
        site_key: The reCAPTCHA sitekey found on the target page.
        page_url: The URL of the page hosting the reCAPTCHA.
        
    Returns:
        The g-recaptcha-response token string, or None on failure.
    """
    
    if not CLIENT_KEY:
        print("Error: CAPSOLVER_API_KEY environment variable not set.")
        return None

    # 1. Create the task
    task_payload = {
        "clientKey": CLIENT_KEY,
        "task": {
            "type": "ReCaptchaV2TaskProxyLess",
            "websiteURL": page_url,
            "websiteKey": site_key
        }
    }
    
    response = requests.post(API_URL, json=task_payload).json()
    if response.get("errorId") != 0:
        print(f"Error creating task: {response.get('errorDescription')}")
        return None
        
    task_id = response.get("taskId")
    
    # 2. Poll for the result
    for _ in range(12): # Poll up to 60 seconds (12 * 5s)
        time.sleep(5) 
        result_payload = {
            "clientKey": CLIENT_KEY,
            "taskId": task_id
        }
        result_response = requests.post(GET_RESULT_URL, json=result_payload).json()
        
        if result_response.get("status") == "ready":
            return result_response["solution"]["gRecaptchaResponse"]
        elif result_response.get("status") == "processing":
            continue
        else:
            print(f"Task failed: {result_response.get('errorDescription')}")
            return None
    
    print("Task timed out.")
    return None
\`\`\`

### 4. Integration Example (Conceptual)

This conceptual example shows how to integrate the solver into your main automation script:

\`\`\`python
# main_automation_script.py
from playwright.sync_api import sync_playwright
from solver_utils import solve_recaptcha_v2

def run_automation():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False)
        page = browser.new_page()
        page.goto("https://target-site-with-recaptcha.com")

        # --- Detection Logic ---
        if page.locator("#recaptcha-iframe").is_visible():
            print("CAPTCHA detected. Initiating resolution...")
            
            # Extract sitekey and URL (replace with actual extraction logic)
            site_key = "6Le-wvkSAAAAAPBii0" # Example sitekey
            page_url = page.url

            # --- Solve Logic ---
            token = solve_recaptcha_v2(site_key, page_url)

            if token:
                # --- Injection Logic ---
                print("Token received. Injecting...")
                # Inject the token into the hidden textarea
                page.evaluate(f'document.getElementById("g-recaptcha-response").value = "{token}"')
                
                # Submit the form
                page.click("#submit-button")
                print("Form submitted successfully.")
            else:
                print("Failed to solve CAPTCHA. Automation halted.")

        browser.close()

if __name__ == "__main__":
    run_automation()
\`\`\`

## 🌐 Advanced Challenge Handling

| Challenge Type | CapSolver Task Type | Integration Note | Documentation |
| :--- | :--- | :--- | :--- |
| **reCAPTCHA v2** | `ReCaptchaV2TaskProxyLess` | Standard sitekey/URL submission. | [reCAPTCHA v2 Guide](https://www.capsolver.com/blog/reCAPTCHA/how-to-integrate-recaptcha-python-data-extraction) |
| **reCAPTCHA v3** | `ReCaptchaV3Task` | Used to generate a high-score token when the AI browser's score drops. | [reCAPTCHA v3 Guide](https://www.capsolver.com/products/recaptchav3) |
| **Cloudflare** | `CloudflareTask` | Requires passing the full page HTML and potentially cookies for context. | [Cloudflare Guide](https://www.capsolver.com/blog/Cloudflare/bypass-cloudflare-challenge-2025) |
| **AWS WAF** | `AwsWafTask` | Specialized task for bypassing AWS WAF challenges. | [AWS WAF Guide](https://www.capsolver.com/products/awswaf) |

## 🤝 Contributing

Contributions are welcome! If you have an improved detection or injection method, please open a pull request.

## 📄 License

This project is licensed under the MIT License.

## 🔗 Resources

*   [CapSolver Dashboard](https://dashboard.capsolver.com/dashboard/overview/?utm_source=github&utm_medium=article&utm_campaign=ai-browser-captcha-solver)- Get your API Key.
*   [CapSolver API Documentation](https://docs.capsolver.com/en/guide/api-server/) - Full list of task types and parameters.
*   [Playwright Documentation](https://playwright.dev/python/docs/intro) - For the AI Browser component.
