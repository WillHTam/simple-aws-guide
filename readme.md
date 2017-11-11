# A Wombat's Guide to Simple AWS Hosting For Fun & Profit
* All text enclosed in the `code blocks` are bash commands
* All certificates/keys you download to communicate with AWS are extremely sensitive. Do not share or expose them to untrusted parties. 
    * Use this tool to search for any high entropy strings (possible keys): https://github.com/dxa4481/truffleHog 
* Delete anything sensitive with `srm filename` to securely remove it
* "30th time's the charm!"

## Preliminary
- Have an AWS account
- Set up an IAM acct for yourself with Admin privileges so you're not always using Root
    - An IAM account underneath Root in the hierarchy can be easily deleted if compromised
- Under 'Network & Security' go to 'Key Pairs'
- Create a key pair with a good name and download it
- Move it to your '~/.ssh' folder & keep it safe
- Make it read-only and only for you with:
    - `chmod 400 your_user_name-key-pair-region_name.pem`

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
- Outbound can be left alone

## Launching the Instance
- Choose your preferred AWI
    - Amazon Linux is good BUT auto-updates
        - RedHat => CentOS based
        - DO NOT USE if you cannot handle auto-updates and easily broken dependencies
            - But I've never encountered any problems yet: remember to use your test/staging servers!
        - ec2 management tools are pre-installed
        - pkg manager is `yum`
    - Ubuntu is production preferred
        - more packages & more answers for problems
        - use `apt-get` or `aptitude`
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
    - `rm /var/www/html/phpinfo.php`
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
### Unchained with Django / MySQL / NGINX / Gunicorn
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
- Finished

## Installing Wordpress
### Famous 20-minute Install
- `wget https://wordpress.org/latest.tar.gz` Get WP
- `tar -xzf latest.tar.gz` and there should be a 'wordpress' folder if you `ls`
- `mysql -u root -p` to access the MySQL terminal
    - See commands here to create non-root db. Be aware of single-quotes vs backticks
    - CREATE USER 'mister-wp-user'@'localhost' IDENTIFIED BY 'yourstrongpassword';
    - CREATE DATABASE \`wp-db-name\`;
    - GRANT ALL PRIVILEGES ON \`db-wp-name\`.* TO "mister-wp-user"@"localhost";
    - FLUSH PRIVILEGES;
    - exit => bye
- `cd wordpress/`
- `cp wp-config-sample.php wp-config.php`
- `vim wp-config.php`
    - define('DB_NAME', 'wp-db-name');
    - define('DB_USER', 'wordpress-user');
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

## Enabling HTTPS with SSL under Apache
### Just a general summary
- Check http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/SSL-on-an-instance.html for an in-depth guide
1. Upload certs and key files to EC2
2. Install mod24_ssl for httpd
3. Update /etc/httpd/conf.d/ssl.conf
4. Update .htaccess in /var/www/html to force all requests to HTTPS
5. Restart httpd with `sudo service httpd restart`

## Finding php.ini
`php -i | grep -i php.ini`

## General Troubleshooting
- If you keep getting time out errors when trying to SSH, check your security group to make sure your IP has permission in Inbound Rules.
- If your WP install is stuck on the 'Maintenance' page, go to the main WP directory and delete the '.maintenance' file. Check for it with `ls -a`
