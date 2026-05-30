import requests
import json
import time
import logging
import os
from typing import Dict, List, Optional
from datetime import datetime
from colorama import init, Fore, Style
init(autoreset=True)

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

class AIAnalyzer:
    def __init__(self, api_key: str = None, provider: str = "grok"):
        self.api_key = api_key
        self.provider = provider.lower()
    
    def analyze(self, data: Dict, target: str) -> str:
        prompt = f"""
You are a senior OSINT intelligence analyst. Provide a professional analysis of the collected data for target: {target}

Include:
1. Key Findings & Patterns
2. Risk Assessment / Red Flags
3. Intelligence Gaps & Recommended Next Steps
4. Executive Summary

Data:
{json.dumps(data, indent=2)}
"""
        try:
            if self.provider == "grok":
                url = "https://api.x.ai/v1/chat/completions"
                model = "grok-beta"
            else:
                url = "https://api.openai.com/v1/chat/completions"
                model = "gpt-4o-mini"
            
            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            }
            
            payload = {
                "model": model,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.6,
                "max_tokens": 1200
            }
            
            resp = requests.post(url, headers=headers, json=payload, timeout=40)
            resp.raise_for_status()
            return resp.json()['choices'][0]['message']['content']
            
        except Exception as e:
            return f"[AI Analysis Unavailable] {str(e)}\nRaw data attached below."


class OSINTFramework:
    def __init__(self, ai_analyzer: Optional[AIAnalyzer] = None):
        self.ai = ai_analyzer
        self.results = {}
        self.session = requests.Session()
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (OSINT-AI-Framework/2.0; Compatible)'
        })

    # ====================== DIRECT MODULES ======================

    def search_username(self, username: str) -> Dict:
        print(Fore.CYAN + f"[+] Checking username: {username}")
        platforms = {
            "Twitter/X": f"https://x.com/{username}",
            "GitHub": f"https://github.com/{username}",
            "LinkedIn": f"https://www.linkedin.com/in/{username}",
            "Instagram": f"https://www.instagram.com/{username}",
            "Reddit": f"https://www.reddit.com/user/{username}",
            "TikTok": f"https://www.tiktok.com/@{username}",
            "Facebook": f"https://www.facebook.com/{username}"
        }
        
        findings = {}
        for name, url in platforms.items():
            try:
                r = self.session.get(url, timeout=10, allow_redirects=True)
                findings[name] = {
                    "url": url,
                    "status": "Active" if r.status_code == 200 else "Not Found / Restricted",
                    "code": r.status_code
                }
            except:
                findings[name] = {"status": "Connection Error"}
            time.sleep(1.2)
        
        self.results['username'] = findings
        return findings

    def search_email(self, email: str) -> Dict:
        print(Fore.CYAN + f"[+] Email reconnaissance: {email}")
        findings = {"email": email, "breach_check": "Manual check recommended (haveibeenpwned.com)"}
        # You can extend with real API calls (Hunter.io, etc.) if you have keys
        self.results['email'] = findings
        return findings

    def search_phone(self, phone: str) -> Dict:
        print(Fore.CYAN + f"[+] Phone number OSINT: {phone}")
        findings = {
            "phone": phone,
            "note": "Use Truecaller, NumLookup, or local carrier lookup tools",
            "international_format": phone
        }
        self.results['phone'] = findings
        return findings

    def search_domain(self, domain: str) -> Dict:
        print(Fore.CYAN + f"[+] Domain reconnaissance: {domain}")
        try:
            whois = requests.get(f"https://api.whois.v2.0/lookup?domain={domain}", timeout=10)
            findings = {"domain": domain, "whois_available": whois.status_code == 200}
        except:
            findings = {"domain": domain, "whois": "Lookup failed"}
        
        self.results['domain'] = findings
        return findings

    def search_ip(self, ip: str) -> Dict:
        print(Fore.CYAN + f"[+] IP address OSINT: {ip}")
        try:
            resp = requests.get(f"https://ipapi.co/{ip}/json/", timeout=8)
            data = resp.json() if resp.ok else {}
            findings = {
                "ip": ip,
                "country": data.get("country_name"),
                "city": data.get("city"),
                "org": data.get("org")
            }
        except:
            findings = {"ip": ip, "status": "Lookup failed"}
        
        self.results['ip'] = findings
        return findings

    def generate_report(self, target: str) -> str:
        report = f"""
╔══════════════════════════════════════════════════════════════╗
║               AI-POWERED OSINT FRAMEWORK v2.0                ║
║ Target: {target:<45} ║
║ Time: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}                  ║
╚══════════════════════════════════════════════════════════════╝
"""

        for module, data in self.results.items():
            report += f"\n[+] {module.upper()} MODULE\n"
            report += json.dumps(data, indent=2) + "\n"
        
        if self.ai:
            print(Fore.YELLOW + "🤖 Running AI Deep Analysis...")
            analysis = self.ai.analyze(self.results, target)
            report += f"\n{'═' * 70}\n🤖 AI INTELLIGENCE ANALYSIS\n{'═' * 70}\n{analysis}\n"
        
        return report


# ====================== MAIN EXECUTION ======================
if __name__ == "__main__":
    print(Fore.GREEN + Style.BRIGHT + "🚀 AI OSINT Framework v2.0 - All Direct Modules Loaded\n")
    
    api_key = os.getenv("XAI_API_KEY") or os.getenv("OPENAI_API_KEY") or input("Enter API Key: ").strip()
    provider = input("Provider (grok/openai) [grok]: ").strip().lower() or "grok"
    
    analyzer = AIAnalyzer(api_key=api_key, provider=provider)
    framework = OSINTFramework(ai_analyzer=analyzer)
    
    target = input("\nEnter target (username, email, phone, domain, or IP): ").strip()
    
    # Auto-route to appropriate modules
    if "@" in target:
        framework.search_email(target)
    elif target.replace("+", "").replace("-", "").replace(" ", "").isdigit():
        framework.search_phone(target)
    elif "." in target and not target.replace(".", "").replace(":", "").isdigit():
        framework.search_domain(target)
    elif target.count(".") == 3 or ":" in target:
        framework.search_ip(target)
    else:
        framework.search_username(target)
    
    # You can call multiple manually if needed:
    # framework.search_username(target)
    # framework.search_domain("example.com")
    
    report = framework.generate_report(target)
    print(report)
    
    filename = f"osint_report_{target.replace('@','_at_')}_{int(time.time())}.md"
    with open(filename, "w", encoding="utf-8") as f:
        f.write(report)
    
    print(Fore.GREEN + f"\n✅ Full report saved as: {filename}")