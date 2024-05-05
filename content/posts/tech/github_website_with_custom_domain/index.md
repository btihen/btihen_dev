---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Github Website with a Custom Domain & SSL"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ["Tech", "website", "ssl", "domain", "github", "cloudflare", "namecheap"]
categories: ["Technology", "Website"]
date: 2020-05-31T13:22:48+02:00
lastmod: 2020-05-31T13:22:48+02:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
### step 0: buy a domain name

For these instructions use the (Namecheap)[https://www.namecheap.com/] service to buy your Domain.

### step 1: point your domain name at: username.github.io (optional)

This takes quite a steps and disables https (more steps follow to renable ssl).  This article got me oriented:
https://dev.to/rightfrombasics/connecting-namecheap-domain-with-github-pages-3nn6

1. log into Namecheap
2. On the left is a sidebar with **Dashboard** and the top.  Click on the **Domain List**
3. Find your domain name and click the **manage** button on the far right.
4. Along the top click on **Advanced DNS**
5. Add your A records to the DNS config.  I typed: `dig btihen.github.io` (of course replace with your github website name) and got:
```
; <<>> DiG 9.10.6 <<>> btihen.github.io
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28239
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;btihen.github.io.		IN	A

;; ANSWER SECTION:
btihen.github.io.	3600	IN	A	185.199.110.153
btihen.github.io.	3600	IN	A	185.199.109.153
btihen.github.io.	3600	IN	A	185.199.108.153
btihen.github.io.	3600	IN	A	185.199.111.153
```
So created the following A Records:
```
type          Host  Value             TTL
A Record      @     185.199.110.153   Automatic
A Record      @     185.199.109.153   Automatic
A Record      @     185.199.108.153   Automatic
A Record      @     185.199.111.153   Automatic
```
6. Then I created a CNAME record:
```
type          Host  Value             TTL
CNAME Record  www   btihen.github.io  Automatic
```

### step 2: configure you github site to accept the domain

You need to make a file called CNAME in the root of your username.github.io repo and it contents must be your new domain name.

For example I used:
```
cd public
touch CNAME
echo 'btihen.me' >> CNAME
git add .
git commit -m 'accept the domain name: btihen.me'
git push
```

### step 3: stop and check

NOW: `http://your-domain-name.com` should work

### step 4: Free ssl for the domain

following the advice from: https://dev.to/rightfrombasics/adding-ssl-to-your-site-free-1fa7

1. create a [cloudflare](https://www.cloudflare.com/) account.
2. choose the dns feature
3. allow cloudflare to scan your dns records (it should get the same results as when you do: `dig username.github.io`)
4. Continue through the cloudflare process & cloudflare will eventually give you 2 nameservers to use.
5. Now you can have cloudflare take over your dns -- log into Namecheap
6. On the left is a sidebar with **Dashboard** and the top.  Click on the **Domain List**
7. Find your domain name and click the **manage** button on the far right.
8. On the top bar choose **Domain**
9. Find the **Nameservers** section
10. Choose **Custom DNS**
11. Add the tow servers given to you by Cloudflare and save.
12. Go back to cloudflare and choose **Full** end to end encryption
13. Choose **Always Use HTTPS**
14. Save and click the **Re-check now** button.

Unfortunately, now you need to wait for a 1/2 hour or morefor the dns service to transfer from Namecheap to Cloudflare.  Theoretically up to 48 hours (but a 1/2 hour is much more typical).
