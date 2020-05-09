![The Cloudbat](https://github.com/WillHTam/simple-aws-guide/blob/master/cloudbat.jpg?raw=true)
# A Wombat's Guide to Simple AWS Hosting For Fun & Profit
* All text in the `code blocks` can be copied as-is, but look for lines you may need to edit to your own details
* All certificates/keys you download to communicate with AWS are extremely sensitive. Do not share or expose them to untrusted parties. 
    * Use this tool to search for any high entropy strings (possible keys): https://github.com/dxa4481/truffleHog 
    * also https://github.com/awslabs/git-secrets
* "30th time's the charm!" - don't be afraid to start over and redo if you mess up
* You will need a text editor if one is not installed (most likely if you are using Arch (why))
    * Install nano, vim, vi, or emacs. I prefer vim.
    * You can easily add syntax highlighting by adding `syntax on` to the `.vimrc` file

## Preliminary
- Have an AWS account
- Set up an IAM acct for yourself with Admin privileges so you're not always using Root
    - An IAM account underneath Root in the hierarchy can be easily deleted if compromised
- Under 'Network & Security' go to 'Key Pairs'
- Create a key pair with a good name and download it
- Move it to your '~/.ssh' folder & keep it safe
- Make it read-only and only for you with:
    - `chmod 400 your_user_name-key-pair-region_name.pem`
- Service or Systemctl?
   - `ps --no-headers -o comm 1`
   - if *systemd* use `systemctl`
   - if *init* use `service`

## Make a VPC
### Enables loading of AWS Resources onto a virtual network
- Go the VPC Wizard
- Make sure your region in the upper right is set to your desired server locations
- Select 'VPC with a Single Public Subnet' & leave the rest as default
- Give it a nice name and choose the correct Availability Zone

## Make a Security Group
### Restrict Access
- Go to EC2 Console
- Make sure you're still in the correct region
- Choose 'Security Groups' & 'Create Security Group'
- Select the VPC you just created
- For the 'Inbound' tab and Sources:
    - Choose HTTP and leave as Anywhere (0.0.0.0/0)
    - Choose HTTPS and leave as Anywhere
    - Choose SSH but DO NOT use Anywhere
        - Select 'My IP' for Source to automatically populate with your current IP
        - Or use http://checkip.amazonaws.com
        - Add your team members by getting their IP, and creating another SSH rule for their IP
            - The format is `theip/32`
- Outbound can be left alone

## Launching the Instance
- Choose your preferred AWI
    - Amazon Linux is good BUT auto-updates
        - RedHat => CentOS based
        - DO NOT USE if you cannot handle auto-updates and easily broken dependencies
            - But I've never encountered any problems yet: remember to use your test/staging servers!
        - ec2 management tools are pre-installed
        - pkg manager is `yum`
    - Ubuntu
        - More packages & more answers for problems, so probably is preferred if you're reading this guide
        - Use `apt-get` or `aptitude`
    - Debian
        - Ubuntu is based on 'Debian testing' and not 'Debian stable', so they will solve vulnerabilities differently and you may have a preference as a result
        - Ubuntu servers are based on Debian, but Ubuntu's packages will be newer and more feature complete. But Debian is lighter, and the only updates you will receive are security updates. Many find Debian more reliable with less downtime as a result.
    - Gentoo
        - you hate yourself
- Choose size
    - t2.micro is free for first year
- Select the correct VPC under Network
- Leave rest as default
    - If you want to prevent accidental deletion, enable 'Termination Protection'
- Enable CloudWatch if you need it
- Add storage will automatically give you 30GB, no action required
- No Tags Needed
- Choose the Security Group you created under 'Existing'
- Choose Launch and choose the key pair you want to match with this instance
- Go to the Instance list and rename the Instance to something recognizable

## Assigning an Elastic IP
### If you don't, all your stuff might disappear when you get reassigned another Public IP
- In the EC2 dashboard, go to 'Network & Security' then 'Elastic IP'
- Use 'Allocate new address'
- No options needed, press OK
- Right click on the newly created IP, select 'Associate Addresses'
- Select the instance and its private IP. Assign.
- DO NOT leave an Elastic IP unassigned, either Assign or Release in a timely manner
    - You will be charged for unused Elastic IPs
- Go back to the Instance List
- Right click the instance and press 'Connect', you will get an SSH command

## Option 1 : LAMP Stack 
### Ready for Wordpress with Amazon Linux / Apache / MySQL / PHP
- Connect to your instance with the SSH command you got in the above step
- `sudo yum update -y`to update all packages
- `sudo yum install -y httpd24 php70 mysql56-server php70-mysqlnd php70-gd`
- `sudo service httpd start` to start the Apache server
- `sudo chkconfig httpd on` will auto-start Apache on instance start
- `chkconfig --list httpd` and 2/3/4/5 should be 'ON'
- From this point onwards, going to the IP should always show an Amazon Linux starter page.
- To allow the ec2-user account to manipulate files in this directory, you must modify the ownership and permissions of the directory. 
- `sudo usermod -a -G apache ec2-user`
- `exit` for changes to take
- reconnect with SSH command
    - run `groups` and the output should be 'ec2-user wheel apache'
- `sudo chown -R ec2-user:apache /var/www` to change /var/www/ ownership to apache group
- `sudo chmod 2775 /var/www` and
- `find /var/www -type d -exec sudo chmod 2775 {} \;` to set write permissions on any future directories
- `find /var/www -type f -exec sudo chmod 0664 {} \;` to add group write permissions
- Testing Permissions (Optional)
    - `echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php`
    - If http://THEIP/phpinfo.php shows a PHP info page, good
    - `rm /var/www/html/phpinfo.php` because you don't want to expose this info.
- Secure the MySQL server
    - `sudo service mysqld start`
    - `sudo mysql_secure_installation`
        - No existing pw so just press Enter
        - Enter a good pw, only use root for admin purposes
        - Use 'Y' for the next four prompts
- `mysql -u root -p` if you need to access the MySQL terminal
- `sudo chkconfig mysqld on` to start the MySQL server on every boot
- Finished

## Option 2: DMNG
### Django / MySQL / NGINX / Gunicorn
- Connect to your instance with the SSH command you got in the above step
- `sudo yum update -y`to update all packages
- `sudo yum install nginx`
- `sudo service nginx start`  The Web Server
- `sudo apt-get install python pip` (Not required if on Amazon Linux I think)
- `sudo pip install django` The Web Application
- `sudo pip install virtualenv`
- `sudo pip install git`
    - You can use git or `scp` to load your project files onto the instance.
- `sudo service mysqld start`
- `sudo mysql_secure_installation`
    - No existing pw so just press Enter
    - Enter a good pw, only use root for admin purposes
    - Use 'Y' for the next four prompts
- `mysql -u root -p` to access the MySQL terminal
    - See commands here to create non-root db. Be aware of single-quotes vs backticks
    - CREATE USER 'mister-user'@'localhost' IDENTIFIED BY 'yourstrongpassword';
    - CREATE DATABASE \`db-name\`;
    - GRANT ALL PRIVILEGES ON \`db-name\`.* TO "mister-user"@"localhost";
    - FLUSH PRIVILEGES;
    - exit => bye
- `django-admin startproject me-project` 'me-project' being the name of the project being created
- `cd me-project/me-project`
- `vim settings.py` / `nano settings.py`
    - under 'INSTALLED_APPS' add name of app e.g. 'me-project'
    - under 'DATABASES=default'
        * ‘ENGINE’: ‘django.db.backends.mysql’,
        * ‘NAME’: ‘db-name',
        * ‘USER’: ‘mister-user’/'Root',
        * ‘PASSWORD’: ‘yourstrongpassword’,
        * ‘HOST’: ‘/var/lib/mysql/mysql.sock’,
        * ‘PORT’: ‘3306’,
    - under 'ALLOWED_HOST' add '*'
- Install django-mysql connectors
    - `sudo yum install mysql-devel`
    - `sudo yum install python-devel`
    - `sudo yum install gcc`
    - `sudo pip install MySQL-Python`
- `sudo pip install gunicorn`
- `gunicorn -b :8000 — env DJANGO_SETTINGS_MODULE=mysite.settings me-project.wsgi &`
- Finished!

## Option 3: PENU
### PM2, Express, Nginx, Ubuntu
- Select the Ubuntu AMI
    - I prefer Ubuntu for this implementation of Nginx, because it is easier to forward the ports (editing nginx/allowed-sites folder)
        - Will likely be contained in `/etc/var/nginx/`
    - If you have more experience with a RHEL/CentOS implementation, then you can stick with Amazon Linux (editing nginx/conf.d)
- Run apt-get update & upgrade
- Install NVM onto Ubuntu
    - `curl https://raw.githubusercontent.com/creationix/nvm/vXXXX/install.sh | bash`
        - Replace the X's with the latest version number
    - Run `nvm install node` and `nvm use node`
        - This will automatically use the latest node, but you can specify a different ver number
        - You can switch between versions of node with `nvm use`
    - You may also require build tools `sudo apt-get install -y build-essential libssl-dev`
- If you were to run `node app.js` on your project now (which can be loaded on with git)...
    - You would would not be able to access it on the main url even if you told express to serve it on ports 80/443 because those are reserved
    - You would instead have to go to `theurl.com:8000` to use your routes after setting in your security group a "Custom TCP" inbound rule for port 8000.
- Nginx will act as a proxy from port 8000 to port 80 (HTTP) or 443 (HTTPS)
    - This is because Linux restricts apps from binding to the first 1024 ports unless they are given root permissions, to prevent malicious processes from binding to sensitive ports such as 80, 443, 22, 21, etc.
    - Nginx will recieve root privileges and therefore be able to handle this problem for us
- First, let us install PM2 to run our node app in the background and automatically on restart
    - `npm install -g pm2`
    - run `pm2 start app.js`
        - Now you can see in PM2's process list that is running much like if you ran 'node app.js'
    - PM2 applications will get automatically restarted, but now we need PM2 to restart on system startup
        - `pm2 startup ubuntu`
            - possibly `pm2 startup systemd`, check with `pm2 startup`
            - this will generate a command that you must run with `sudo`
    - check processes with `pm2 status`
    - see your console logs and outputs with `pm2 log <PROCESSNAME> --lines 200`
- Let's use Nginx
    - `sudo apt get update`
    - `sudo apt get install nginx`
    - `sudo vim /etc/nginx/sites-available/default`
        - 'sites-enabled' is symlinked to this folder
    - Add this
        ```
        server {
            listen 80;
        
            server_name sitename.com;
        
            location / {
                proxy_pass http://127.0.0.1:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
            }
        }
        ```
    - Replace 'sitename' with your domain and 3000 with the port you are serving on if necessary
    - `sudo nginx -t` to check if the new edit is free of errors 
    - `sudo systemctl restart nginx`

## Option 4: AWS S3
### A good option for a static website
- If your app is a simple static site, then a site hosted on AWS S3 should be considered
  - Low upkeep cost
  - Fairly easy to set up, but does require tangling with the many AWS dashboards
- Compile your React/whatever app
  - ensure that the entry point to the app is through index.html
- Create 2 S3 buckets
  - They *must* be named according to your site: one of `site.com` and `www.site.com`. Likewise if you are establishing subdomains: `www/subdomain.site.com`
  - On creation, ensure that the access permissions are open and public
- Upload to S3
  - I find it's easier to use the command line to upload from your build folder
  - `aws s3 cp /localpath/ s3://<bucket name> —-recursive`
- In Properties -> Static Site Hosting for the bucket without `www`, select the 'Use this bucket for hosting option', and set index.html for the index document
  - I have frequently encountered 404 Missing Key errors from AWS, which were remedied by simply making index.html the file to be accessed for the 'Error dcoument'
- For the bucket with `www`, under the same menu set to 'Redirect requests' with http for the second field. Later I will discuss how to use https.
- Under Permissions -> Bucket Policy, we will give both buckets the standard open website policy
```
{
    "Version": "2008-10-17",
    "Id": "PolicyForPublicWebsiteContent",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<sitename.com>/*"
        }
    ]
}
```
- Note: ensure that the three colons prior to the sitename and after s3 persist in the value for the "Resource" key.
- If you are hosting a website without SSL, you can end here.  Get the endpoint from the Properties->Static Site Hosting menu and ensure that the site works by going to the address.
  - If it works, you can proceed to make a CNAME record to the www site with your DNS' cPanel.
### Enabling HTTPS
- The following is for certificates bought on other sites
  - Consider using EFF's Certbot which is both free and very easy to use, but I'm not sure if they can be registered with AWS
- Go to AWS Certificate Manager, and select Import Certificate. Button located at the top of the page.
  - Copy the required information into the fields
  - For certificates from ssl.com, there is a option to download Amazon certificates, which is the one you want
- Go to Cloudfront, and make a new distribution
  - Set the Origin as the S3 bucket, set 'Redirect HTTP to HTTPS', and select the certificate you imported.
- Change the CNAME record to point at the CloudFront url, shown when you click the corresponding site on the CloudFront dashboard
  

## Redirecting your Site
- Now that you've got your instance set up, you obviously don't want people to put in an IP Address
- Get a domain name from namecheap.com / bluehost.com / godaddy.com or wherever you'd like
    - The terms and processes used here will be the same for any website, but they will be organized in different ways
- Now that your ec2 instance is set up as you like, copy the instance's IP.
- Go to Route53, and select 'Hosted Zones'
- Select 'Create Hosted Zone', and enter your purchased domain name, comments not required, and 'Public Hosted Zone'
- Copy all the nameserver entries (of which there should be 4) in the NS set for later
- Now we need to create two A records
    - Go to 'Create Record Set'
    - Leave 'name' field blank
    - Type as "A - IPv4...."
    - 'Value' as the instance's IP you copied earlier
    - Then finish with 'Create'
    - Repeat again, but this time put 'www' in 'name'
        - This will make 'yourdomain.com' and 'www.yourdomain.com' registered under the nameservers
- For Namecheap, go to your domain's Manage -> Domain menu, then Nameservers.
- Change 'NameCheap DNS' to 'CustomDNS', and put each of the nameserver entries you copied from earlier on their own line
- AWS puts a period on the end of each, remove them
- Click 'Save All'
- DNS Propagation should occur with 25 minutes, maximum of 24 hours.

- GitLab Pages, go to Settings -> Pages, and enter your domain name with and without www.
- Then go to your CPanel, and instead of editing the nameservers, go to DNS settings (under Advanced DNS for nameCheap)
- Then set A records that point to the GL Pages IP.

## Installing Wordpress
### Famous 20-minute Install
- `wget https://wordpress.org/latest.tar.gz` Get WP, place where you like but most likely '~'
- `tar -xzf latest.tar.gz` and there should be a 'wordpress' folder if you `ls`
- `mysql -u root -p` to access the MySQL terminal
    - See commands here to create non-root db. Be aware of single-quotes vs backticks
    - CREATE USER 'mister-wp-user'@'localhost' IDENTIFIED BY 'yourstrongpassword';
    - CREATE DATABASE \`wp-db-name\`;
    - GRANT ALL PRIVILEGES ON \`wp-db-name\`.* TO "mister-wp-user"@"localhost";
    - FLUSH PRIVILEGES;
    - exit => bye
- `cd wordpress/`
- `cp wp-config-sample.php wp-config.php`
- `vim wp-config.php`
    - define('DB_NAME', 'wp-db-name');
    - define('DB_USER', 'mister-wp-user');
    - define('DB_PASSWORD', 'yourstrongpassword');
- Under 'Authentication Unique Keys and Salts'
    - Replace the blank values with https://api.wordpress.org/secret-key/1.1/salt/
- `cp -r wordpress/* /var/www/html/`
- Allow WP to use Permalinks
    - `sudo vim /etc/httpd/conf/httpd.conf`
        - Change AllowOverride from 'None' to 'All'
- `sudo chown -R apache /var/www` Change the file ownership of /var/www and its contents to the apache user.
- `sudo chgrp -R apache /var/www` Change the group ownership of /var/www and its contents to the apache user.
- `sudo chmod 2775 /var/www`
- `find /var/www -type d -exec sudo chmod 2775 {} \;`
- `find /var/www -type f -exec sudo chmod 0664 {} \;`
- `sudo service httpd restart`
- Done

## Enabling HTTPS with SSL
### Bought Your Certificates
- upload to the server with scp
```
scp -i "keypair" -r /path/to/stuff ec2servername:/path/to/target/dir
```
- copy to an appropriate location

### NGINX
```
sudo mkdir /etc/nginx/ssl
sudo cp server.crt /etc/nginx/ssl
sudo cp server.key /etc/nginx/ssl
```
- edit default file in sites-available
- delete the listen 80 lines
- uncomment the listen 443 lines
- add this below the listen 443 lines
```
ssl on;
ssl_certificate /etc/nginx/ssl/your_domain_name.pem; 
ssl_certificate_key /etc/nginx/ssl/your_domain_name.key;
```
- don't forget the force to https line below, added beneath the ssl block
- `sudo systemctl restart nginx`

### APACHE
- edit httpd.conf, likely in `/etc/apache2/httpd` or `/etc/httpd/httpd.conf` or `/etc/httpd/conf.d/ssl.conf`
```
# not consecutive lines
SSLCertificateFile /pathto/certificate.crt
SSLCertificateKeyFile /pathto/keyfile.key
```
or
```
<VirtualHost 192.168.0.1:443>
    DocumentRoot /var/www/html2
    ServerName www.yourdomain.com
        SSLEngine on
        SSLCertificateFile /path/to/your_domain_name.crt
        SSLCertificateKeyFile /path/to/your_private.key
        SSLCertificateChainFile /path/to/cabundle.crt
    </VirtualHost>
```
- test configuration with `apachectl configtest`
- restart apache `sudo service httpd restart`

### Free!
- https://certbot.eff.org
- if need to add more sites (add www and subdomains)
`certbot certonly extend -d site1.com -d site2.com`
- Needed to delete 'return 404' line added by certbot
- On nginx, need to force to https by changing last bracket to
```
server {
    listen 80;
    
    server_name _;

    return 301 https://$host$request_uri;
}
```

## Setting Ubuntu Locale
- `locale -a`
- 'sudo locale-gen en_SG.UTF-8`
- `LANG=en_SG.UTF-8`
- `update-locale LANG=en_SG.UTF-8`
- `cat /etc/default/locale`

## Finding php.ini
`php -i | grep -i php.ini`

## General Troubleshooting
- If you keep getting time out errors when trying to SSH, check your security group to make sure your IP has permission in Inbound Rules.
- If your WP install is stuck on the 'Maintenance' page, go to the main WP directory and delete the '.maintenance' file. Check for it with `ls -a`
