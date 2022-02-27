# How to Host CTFd on AWS Lightsail
This guide details the process for setting up a lightsail instance with a static IP in AWS, configuring CTFd inside of a docker container on the instance, using AWS CloudFront to serve as a CDN for the website which includes SSL/TLS certificates/encryption, and finally configuring a custom domain to access your new CTF!

Prerequisites: An AWS account, and approximately $6-20 in monthly AWS costs, optional costs include approximately $10-20/year for a custom domain name

## Table of Contents
### Part 1: Set up Lightsail and a Static IP
### Part 2: Configure CTFd on the Instance
### Part 3: Set up a Lightsail CloudFront Distribution (optional)
### Part 4: Configuring a Custom Domain Name (optional)
### Part 5: Configure the Firewall and Set Up a Reverse Proxy Using Nginx

Resources: <br>
AWS Lightsail:  https://lightsail.aws.amazon.com <br>
AWS Route53:    https://console.aws.amazon.com/route53 <br>
CTFd Github:    https://github.com/CTFd/CTFd <br>
CTFd Docs:      https://docs.ctfd.io/docs <br>

#

### Part 1: Set up Lightsail and a Static IP

1. Navigate to AWS Lightsail at https://lightsail.aws.amazon.com
2. Click "Create Instance" 
3. Click "OS Only" and select "Ubuntu 20.04 LTS"
4. Scroll down and select the hardware for your instance <br>
- Note: CTFd Docs recommends a minimum of dual core CPU and 1 GB RAM. I've had no issues running an instance with 2 GB RAM and 1 vCPU at $10/month, however if you're expecting very little traffic a less expsnive option may work, and if you're expecting a lot of traffic, a more expensive option may be better
5. Identify your instance with a unique name and click "Create-Instance"
6. While that boots, click on "Networking" within the Lightsail Console
7. Click "Create Static IP"
8. Select the instance you've just created to easily attach the IP, then name your static IP and click "Create"
9. Go back to instances and click on the name of your new instance, then go to the Networking tab and under IPv4 Firewall add a custom TCP rule for port 8000, leave the checkbox checked for duplicating the rule for IPv6
10. By now your instance should be booted so head back to instances again and click the >_ icon, you should be greeted by a command line console for your instance


### Part 2: Configure CTFd on the Instance

1. Here we can more or less follow the instructions in the CTFd Docs at https://docs.ctfd.io/docs/deployment/installation
2. Run the following commands:
```
sudo apt-get update
sudo apt-get upgrade
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
7. You should now be able to connect to your new CTF for the first time! Navigate to `http://<machine-ip>:8000` and walk through the steps on th webpage to setup your CTF structure 
- Note: Adding, editing, and deleting users, challenges, pages, etc. is all done through the Admin Panel on the website, so once everything is fully configured within AWS there's very little need to ssh back into the lightsail instance, and therefore it's fairly user friendly for those who are not web-development experts

### Part 3: Set up a Lightsail CloudFront Distribution (optional)

### Part 4: Configuring a Custom Domain Name (optional)

### Part 4: Configure the Firewall and Set Up a Reverse Proxy Using Nginx

1. Run the following commands to add some firewall rules, and then enable the firewall:
```
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'

sudo ufw enable
```

2. You may have noticed that navigating to your machine IP on port 80 gives you the default Nginx page, here is where we configure Nginx as reverse proxy to point port 80 to the correct location
3. Navigate to /etc/nginx/sites-available and create a file in there with a file name of your domain name, i.e. example-ctf.com
4. Use your favorite text editor to open up the file and copy and paste this in there, please note, a few lines down you need to change <your-domain.com> to your actual domain name (if you chose to purchase one), as well as your CloudFront distribution default domain, if you chose not to purchase a custom domain, just put the CloudFront default domain:

```
limit_req_zone  $binary_remote_addr zone=mylimit:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=addr:10m;
server {
	server_name <your-domain.com> <CloudFront distro default domain>;
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
5. Next, create a symbolic link of this file to the nginx sites-enable directory using this: `sudo ln -s /etc/nginx/sites-available/your-domain.com /etc/nginx/sites-enabled/your-domain.com`
6. Reload nginx: `sudo nginx -s reload`

