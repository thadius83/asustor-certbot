Updated 27/05/2024 to work with ADM 4.x as some locations have change.

Requires Python to be installed either via CLI or the ADM interface
Verified working on ASUSTOR 6706T ADM 4.3.0.RSB1

I'm sure it can be done cleaner and more automated but this was simple.

This particular setup utilises cloudflare dns challenge + API key for certificate validation

# asustor-certbot
Automated [Let's Encrypt](https://letsencrypt.org/) certificate renewal via [certbot](https://certbot.eff.org/docs/) on an Asustor NAS box

**Note:** this project no longer recommends attempting to use certbot on an Asustor NAS due to the increasing difficulties with [certbot installation on an Asustor NAS](https://github.com/bebo-dot-dev/asustor-certbot/blob/master/CertbotInstallation.md). Alternative options include the Asustor App Central installable "Let's Encrypt ACME Client" app (a wrapper around [https://github.com/acmesh-official/acme.sh](https://github.com/acmesh-official/acme.sh)) and other options that are listed [here](https://letsencrypt.org/docs/client-options/) on the letsencrypt website.

Why does this even exist when there's a certificate management built into the ADM (Asustor Data Master) linux o/s? Here are a few reasons.

#### The cons of doing certificate management the Asustor way: ####

1. The behaviour of the ceritificate manager built into ADM is dependent on this Asustor supplied binary: `/usr/builtin/bin/certificate`. For automated certificate renewal, ADM sets up a crontab job that looks like this: `0 0 * * * TAG=CERTIFICATE /usr/builtin/bin/certificate update-cert` so that this process is run continually daily at midnight.

   What does this process do? It renews certificates when all of the requirements for successful certificate renewal are correctly aligned.
   
   What happens when certificates fail to renew? The process fails silently. Asustor support were unable to describe what happens when certificate renewal fails other than to say _"logging of SSL certificate renewal errors is not enabled"_. `/usr/builtin/bin/certificate` appears to be an undocumented Asustor binary that has unknown behaviour other than it being confirmed by Asustor that **nothing is logged anywhere** when certificate renewal fails.
   
2. Successful certificate renewal via the `/usr/builtin/bin/certificate` binary is dependent on a web service listening on the NAS on port 80. **Port 80 needs to be permanently open to the NAS** and a web server needs to be running on the NAS **listening on port 80 at all times** (typically apache on an Asustor NAS) in order for certificate renewal to succeed. This is a security risk.

   What other requirements are there for successful certificate renewal to occur via `/usr/builtin/bin/certificate`? Other than port 80 needing to be being permanently open, the requirements are unknown because this binary is undocumented and poorly supported by Asustor.

3. There's lack of flexiblity when relying on `/usr/builtin/bin/certificate` to perform certificate renewal. Since this binary is undocumented, there is **no known way to perform any additional required actions when certificates fail to renew or do renew successfully**.

#### The pros of doing certificate management the [certbot](https://certbot.eff.org/docs/) way: ####

1. certbot is [well documented](https://certbot.eff.org/docs/).
2. certbot logs everything it does including renewal failure and success.
3. certbot is chainable / extendable. It is possible to perform additional required actions when certificates fail to renew or renew successfully via regular shell scripts.
4. when certbot runs in 'standalone' or 'renew' mode with a http challenge, it spins up a temporary webserver on demand and it is possible to specify which custom port that the LetsEncrypt CA should perform its http callbacks on. Certbot's temporary webserver will always listen on port 80 or port 433 however the CA http callback custom port can be port forwarded / mapped to port 80 / 443.
5. certbot supports obtaining and renewing certificates over an ssl connection.
6. certbot supports dns challenge if that method is preferred over http challenge.
7. certbot is open: https://github.com/certbot/certbot.

#### Installing certbot on the NAS ####
1. certbot is written in python, install python from App Central and verify the install `python --version`.
2. once you have python installed check you have pip installed i.e. `pip --version`.
3. install certbot with pip i.e. `pip install cryptography` && `pip install certbot`.
4. create Cloud Flare API key with edit zone permissions
5. mkdir /usr/builtin/etc/letsencrypt/
6. create cloudflare.ini in /usr/builtin/etc/letsencrypt/
7. add API key to cloudflare.ini
8. chmod 600 /usr/builtin/etc/letsencrypt/cloudflare.ini
9. pip / pip3 install certbot-dns-cloudflare
10. Run the initial generation of the cert - /usr/local/AppCentral/python3/bin/certbot certonly --config-dir /volume0/usr/builtin/etc/letsencrypt --dns-cloudflare --dns-cloudflare-credentials /usr/builtin/etc/letsencrypt/cloudflare.ini -d hostname.domain.com --dns-cloudflare-propagation-seconds 60 --agree-tos -m user@email.xyz
11. deploy nas-certbot-renewal.sh to /volume0/usr/builtin/bin
12. deploy nas-certbot-deploy.sh to /volume0/usr/builtin/etc/letsencrypt/renewal-hooks/post/
13. chmod +x nas-certbot-renewal.sh & nas-certbot-deploy.sh
14. change SOURCE_CERT=/live/your-cert-domain.net to your domain in nas-certbot-deploy.sh
15. Add the following line to crontab -e : 0 2 * * * /usr/builtin/bin/nas-certbot-renewal.sh

Note that depending on your Asustor NAS box model and the available tools installed on the NAS box, Certbot installation might not be straightforward. See further details [here](https://github.com/jjssoftware/asustor-certbot/blob/master/CertbotInstallation.md).



#### Shell scripts included in this github repo ####
1. nas-certbot-renewal.sh

   This is a certbot [Let's Encrypt](https://letsencrypt.org/) certificate renewal script. It simply calls certbot to perform certificate renewal for any certificates previously created by certbot. This script can be crontab scheduled. Further details are in the script.

2. master/nas-certbot-deploy.sh

   This is a certbot renewal-hooks/deploy script. This script is called by certbot when certificate renewal succeeds. Any custom actions that need to be performed upon successful certificate renewal can be included in a renewal-hooks/deploy script. Further details are in the script.
