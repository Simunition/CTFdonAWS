# How to Host CTFd on AWS Lightsail
This initial post has been upgraded to be hosted on my personal blog which you can find at https://simscyberops.com.

This guide details the process for setting up a lightsail instance with a static IP in AWS, configuring CTFd inside of a docker container on the instance with Nginx acting as a reverse proxy, using AWS CloudFront to serve as a CDN for the website, and finally configuring a custom domain with SSL/TLS certificates to access your new CTF!

Prerequisites: An AWS account, and approximately $6-20 in monthly AWS costs, and a few hours of your time. Optional costs include approximately $10-20/year for a custom domain name

## Table of Contents
### Part 1: Set up a Lightsail Instance and a Static IP
### Part 2: Configure CTFd on the Instance
### Part 3: Set up a Lightsail CloudFront Distribution (optional)
### Part 4: Configure the Firewall and Set Up a Reverse Proxy Using Nginx
### Part 5: Configuring a Custom Domain Name (optional)

Resources: <br>
AWS Lightsail:  https://lightsail.aws.amazon.com <br>
AWS Route53:    https://console.aws.amazon.com/route53 <br>
CTFd Github:    https://github.com/CTFd/CTFd <br>
CTFd Docs:      https://docs.ctfd.io/docs <br>

- Note: Parts 3 and 5 are optional if you don't care about having a custom domain name, or encrypting communcation with SSL/TLS to host your site over HTTPS. Alternatively, if you're hosting any other type of website and have it set up and running over HTTP on lightsail, parts 3 and 5 can be followed to add that same functionality to any lightsail hosted websites. 

#

### Part 1: Set up a Lightsail Instance and a Static IP

1. Navigate to AWS Lightsail at https://lightsail.aws.amazon.com
2. Click "Create Instance" 
3. Click "OS Only" and select "Ubuntu 20.04 LTS"
4. Scroll down and select the hardware for your instance
- Note: CTFd Docs recommends a minimum of dual core CPU and 1 GB RAM. I've had no issues running an instance with 2 GB RAM and 1 vCPU at $10/month, however if you're expecting very little traffic a less expsnive option may work, and if you're expecting a lot of traffic, a more expensive option may be better
5. Identify your instance with a unique name and click "Create-Instance"
6. While that boots, click on "Networking" within the Lightsail Console
7. Click "Create Static IP"
8. Select the instance you've just created to easily attach the IP, then name your static IP and click "Create"
9. Go back to instances and click on the name of your new instance, then go to the Networking tab and under IPv4 Firewall add a custom TCP rule for port 8000, leave the checkbox checked for duplicating the rule for IPv6 and click the green button to add the rule
10. By now your instance should be booted so head back to instances again and click the >_ icon, you should be greeted by a command line console for your instance


### Part 2: Configure CTFd on the Instance

1. Here we can more or less follow the instructions in the CTFd Docs at https://docs.ctfd.io/docs/deployment/installation, however, I had to add in a few to avoid some errors along the way
2. Run the following commands:
```
sudo apt-get update
sudo apt-get upgrade #(optional but recommended)
sudo apt-get install docker
sudo apt-get install docker-compose
sudo apt-get install nginx 
sudo apt-get install ufw 

sudo systemctl stop nginx

git clone https://github.com/CTFd/CTFd.git
cd CTFd
```
3. Inside the CTFd folder there's a file called docker-compose.yml, open that up with your favorite text editor and under services > environment: you're going to add a new environment variable called `SECRET_KEY` and give it a random string i.e. `SECRET_KEY=Rd42RfGayKozvU2DBfsC` 
4. Still within the file, find the variable called WORKERS and change it from 1 to somewhere between about 4 and 10, the more users you expect, the higher it should be, but also more taxing on the system. Once it's added save and exit the text editor
6. Now, from within the CTFd folder that contains docker-compose.yml, run:
```
sudo docker-compose up 
```
- Note: This step will take a while to run, once it's complete use `sudo lsof -i -P -n | grep LISTEN` to check your open ports, if both 80 and 8000 both say docker-pr then exit out of the console and use the lightsail console to stop the machine, then start it again and wait a few minutes for it to boot up. Once it's back on connect using SSH and run `sudo lsof -i -P -n | grep LISTEN` again, if it's adjusted to show nginx on port 80 and docker-pr on 8000, you're good to continue
7. You should now be able to connect to your new CTF for the first time! Navigate to `http://<machine-ip>:8000` and walk through the steps on the webpage to setup your CTF structure 
- Note: Adding, editing, and deleting users, challenges, pages, etc. is all done through the Admin Panel on the website, so once everything is fully configured within AWS there's very little need to ssh back into the lightsail instance, and therefore it's fairly user friendly for those who are not web-development experts

### Part 3: Set up a Lightsail CloudFront Distribution (optional)

AWS CloudFront is a content delivery network (CDN), which is a system that can be used to cache static portions of webpages for faster service to clients around the world. CloudFront acts as a middle man between clients and our website so when someone reaches out CloudFront sends what they're asking for from cached content. If it doesn't have the content cached (such as with dynamic content) it reaches out and requests the information from our lightsail instance acting as a web server. This also adds a nice layer of security as clients are not directly accessing your instance, instead accessing CloudFront who pulls from your instance on their behalf.

1. On the AWS Lightsail console navigate to the "Networking" tab, please note this is not the same as the Networking tab within an instance, you should see buttons for Create static IP, Create DNS zone, etc.. 
2. Click on "Create Distribution"
3. Select your Lightsail instance as the origin
4. Under Caching behavior select the preset "Best for dynamic content"
5. Choose your distribution plan based on how much traffic you antiticpate for your CTF, ensure your selection is within your budget, the option for 50Gb/month is currently free for the first year and $2.50/month thereafter
6. Name your distribution and click "Create"
7. The distribution Status will show "In Progress" for a while as it caches the website, in the meantime the next step is to congifure an Nginx reverse proxy using your distribution default domain, take note of what it is by looking at the top right hand side of the management page 
- Note: If you click the link for the default domain right now it will resolve to your default nginx page, we'll fix that in the next step


### Part 4: Configure the Firewall and Set Up a Reverse Proxy Using Nginx

1. Jump back on youre lightsail instance running CTFd and run the following commands to add some firewall rules, and then enable the firewall:
```
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'

sudo ufw enable
```

2. You may have noticed that navigating to your machine IP on port 80 gives you the default Nginx page, here is where we configure Nginx as reverse proxy to point port 80 to the correct location
3. Navigate to /etc/nginx/sites-available and create a file using sudo in there for your CTF, i.e. `sudo touch example-ctf`
4. Use your favorite text editor along with sudo to open up the file and copy and paste the text below in there, please note, a few lines down you need to replace the variable for your cloudfront distro default name with your actual distributions name (without the https://), it should look something like `d*******.cloudfront.net`
- Note: If you're configuring a custom domain for your website you'll add it to the server name variable later on, to list multiple server names you separate them with a single space, i.e. `server_name dsomething.cloudfront.net your-domain.com`
- Another Note: This file is also used to limit the rate of requests to your web server which can help protect against certain types of attacks with high request rates, a number of variables in here prevent too many concurrent connections from the same host, as well as the number of requests per second that are allowed

```
limit_req_zone  $binary_remote_addr zone=mylimit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=addr:10m;
server {
	server_name <CloudFront distro default domain>;
	limit_req zone=mylimit burst=15;
	limit_conn addr 10;
	limit_req_status 429;
	client_max_body_size 8M;
	location / {
    		proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
  }
}
```
5. Next, create a symbolic link of this file to the nginx sites-enable directory using the command: `sudo ln -s /etc/nginx/sites-available/<your-ctf-file> /etc/nginx/sites-enabled/<your-ctf-file>`
6. Reload nginx: `sudo nginx -s reload`
7. If you go back to your browser and navigate to your CloudFront distributions default domain it should now resolve to your CTF home page!
- Note: if you don't want to purchase a custom domain, or don't care about setting up a custom domain to access your CTF, you can stop here and use the CloudFront default domain to access your CTF page

### Part 5: Configuring a Custom Domain Name (optional)

Preface: Custom Domain's can be easily purchased through AWS, and you can use AWS to obtain SSL/TLS for certificates for free, however, purchasing a domain costs a minimum of about $5/yr, and some even cost thousands or hundreds of thousands of dollars, that said, if you're not picky about the name many .com, .org, and .net domains are fairly affordable at about $10-12/yr.

1. Go back to the main AWS console (not Lightsail) and search for Route 53, once you get to the dashboard you should see some statistics on your assets and buttons for other functions in the service, on the left hand side find "Registered domains" and click it
2. In here, click "Register Domain"
3. Here is where you'll search for what domain's are available and pick one that you like (and is within your price range), once you find one click "Add to cart" and "Continue"
4. On this page you have to put in Registrant Contact information, at the bottom there's a Privacy Protection section, if you're putting in your personal information and not company contact information ensure "Enable" is checked, this obfuscates your personal data that is reported to ICANN to minimize exposure, for example, your email may appear on ICANN as something like `owner-1232923@<your-domain>.whoisprivacyservice.org`
5. Click Next, confirm your information, decide whether or not you want your domain to automatically renew (once a year), and then agree to the registrations agreemnt and complete your order
- Note: Some types of domains require verification of your email, if that's the case for yours it will be obvious on the page and require completion of that verification before you're able to complete purchase
7. It takes a while for the domain registration to take so in the meantime head back to the lightsail console for some related tasks
8. Go to the management console for your Lightsail CloudFront Distribution and click on the "Custom domain" tab
9. Near the bottom click "+ Create Certificate"
10. Enter your domain name and click "Create"
11. That will show pending for a few seconds, then it will state "Validation in Progress," take note of the section near the bottom that shows information for a CNAME record, we're going to use that later to validate that you own the domain so that the certificate can be issued
12. Go back to Route 53 and check to see if your domain registration has completed, this process can take anywhere from a few minutes to several hours, so you may have to step away and come back after some time has passed 
13. Welcome back! When you registered your domain AWS automatically created a hosted zone for it with 2 records, on the left hand side of Route 53 click "Hosted Zones" and then find your hosted zone for your new domain and click it
14. In here you should see the 2 pre-created records, we're going to create 2 more, one to validate your TLS/SSL certificate, and one to point your domain to your CloudFront distribution
15. Click "Create Record" 
16. In the center change record type to CNAME, if you remember, the TLS/SSL certificates we started earlier had information for a CNAME record, that's what it's waiting on for validation
17. From the Lightsail management console find that CNAME information on your CloudFront distribution and copy the "Name" value, back in the Route 53 Create Record page paste it into the Record name box, make sure it doesn't repeat your domain at the end, it should be `<_long string>.your-domain.com/net/org/whatever` NOT `<_long string>.your-domain.com.your-domain.com`
18. Next copy the value from the Lightsail tab into the Value in Route 53 create record, this should be something like `_<long string>.<another string>.acm-validations.aws.`
19. Leave everything else default and click "Create Records"
20. Once again can take anywhere between a few minutes and several hours to finish validating the certificate, complete the next few steps and if you hit step 32 and it's still not complete, take a break and come back
21. In the mean time we're going to create one more record in your hosted zone
22. Copy your Lightsail CloudFront distributions default domain 
23. In the route 53 hosted zone for your domain, click "Create Record" again
24. Leave the record name completely empty and on the right hand side of the page toggle the switch for "Alias"
25. Underneath that there's a dropdown for choose endpoint, click that and select "Alias to CloudFront distribution"
26. This part is a bit deceiving because it says "Choose distribution" but most likely your distro won't appear in the resources list, instead you're going to paste the default domain into that box, as a note here, make sure the https:// and trailing / both get removed, the domain should just be something like `d*****.cloudfront.net`
27. Click "Create records", leave all the other boxes and dropdowns empty or whatever was there by default
28. We have one last configuration to make inside of the Lightsail instance running CTFd so go back to the Lightsail console and hop into the command line for your instance
29. Use your favorite text editor and sudo privileges to jump into a file you created earlier that starts with /etc/nginx/sites-available/<your-file>
30. Add your new custom domain name to the server_name variable, separate the two by just using a single space, i.e. `server_name dsomething.cloudfront.net your-domain.com` 
31. Run `sudo nginx -s reload` then exit the console
32. Once the SSL/TLS certificate validation is complete in the Lightsail console there should be a switch for custom domains above that unlocks, click that switch to enable custom domains
33. This will lock the distribution for a few minutes and the Status at the top will reflect "In progress"
34. When the distribution changes back to Status: Enabled you're done! if you navigate to `your-domain.com/org/net/whatever` or `https://your-domain.com/org/net/whatever` you should be greeted with the front page of your new AWS hosted CTF! 

## Conclusion 

CTFd is an incredible platform built out by Kevin Chung and his team, now that you've made it here and your AWS environment is set up to host your webpage, you can do almost all, if not all your administration from inside the website. You'll notice an icon at the top that looks like a wrench that's called "Admin Panel", click it and from there you can easily add or edit challenges, users, etc.. as well as view statistics on challenge solves (even failed attempts), you can see any user or teams score currently over the course of time. Even the possibilities when setting up challenges is extremely extensive. Read more about what you can do and how to do it on the CTF docs website at:  https://docs.ctfd.io/docs.
