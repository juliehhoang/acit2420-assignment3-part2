# ACIT 2420 Assignment 3 Part 2

This outlines the setup process for the webgen system and nginx web server which is deployed across two servers using a load balancer.

## Creating Load Balance and Servers
- Firstly you will need to create the servers that the load balancer with serve
1. In DigitalOcean click "Create" and then "Droplets"
2. The region should be San Francisco with datacent SF03
3. The image should be Arch Linux
4. In the details the quantity should be 2 droplets
5. The hostname can be whatever you want
6. In tags create the tag web
7. Click "Create Droplet"
- Now to create the load balancer
1. The region should be San Francisco with datacent SF03 (same as the servers)
2. Make sure it is external(public)
3. Connect it to the "web" tag so that it load balances the servers with the "web tag" (the two servers you created)

- Once you are done you can ssh into each by copying the ip address:
```ssh -i .ssh/do-key arch@your-droplets-ip-address```

- Next, start by cloning the repository into both your servers
```
git clone https://github.com/juliehhoang/acit2420-assignment3-part2.git
```

- The webgen system on both servers should look the same and will create the following directory structure:
```
/var/lib/webgen/
├── bin/
│   └── generate_index
└── HTML/
    └── index.html
```

- The `generate_index` script will need to be moved into `/var/lib/webgen/bin/` (to be seen later) to generates the HTML.
- The `HTML` directory contains the `index.html` file (generated by `generate_index`), which is served by `nginx`.

**note:**
Both servers should be identical except for the files that will serve some test files on both servers

## Creating the System User

To enhance security and prevent accidental system changes, the `webgen` user is created to run the script instead of using a regular user or root. Using a system user minimizes the risk of security vulnerabilities and ensures that only the required permissions will be granted.
```bash
sudo useradd -r -s /bin/bash webgen
```
## Setting Up Directories and Permissions
- You will need to create the directories:
```bash
sudo mkdir -p /var/lib/webgen/bin
sudo mkdir -p /var/lib/webgen/HTML
```
- The webgen user needs the proper permissions to read/write to the bin and HTML directories. 

```bash
sudo chown -R webgen:webgen /var/lib/webgen
sudo chmod u=rwx,g=rx,o= /var/lib/webgen /var/lib/webgen/HTML
```

## Moving the `generate_index` Script
- The generate_index script should be placed in /var/lib/webgen/bin/. This script generates system information in HTML.
- You can move it to `/var/lib/webgen/bin/` using:
```bash
sudo mv generate_index /var/lib/webgen/bin/
```
- You will need to give it the correct ownership and permissions:
```bash
sudo chown webgen:webgen /var/lib/webgen/bin/generate_index
sudo chmod +x /var/lib/webgen/bin/generate_index
```

## Configuring the File Server
- You will first need to create the `documents` directory and add test files:
```bash
sudo mkdir -p /var/lib/webgen/documents
```
- Now you can move the text files into the directory:
```bash
sudo mv file-one /var/lib/webgen/documents
sudo mv file-two /var/lib/webgen/documents
```
There are tests provided for both servers
- Make sure they have the right permissions:
```bash
sudo chown -R webgen:webgen /var/lib/webgen/documents
sudo chmod u=rwx,g=rx,o= /var/lib/webgen/documents
sudo chmod u=rw,g=r,o= /var/lib/webgen/documents/file-one
sudo chmod u=rw,g=r,o= /var/lib/webgen/documents/file-two
```

## The `systemd` Service and Timer
To automate the generation of the `index.html` file, we use `systemd` timers. The generate_index service is triggered by a timer that runs daily.

- Move the service and timer files
into the correct directories:
```bash
sudo mv generate-index.service /etc/systemd/system/
sudo mv generate-index.timer /etc/systemd/system/
```
- Now you can enable and start the timer:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now generate-index.timer
```
- Make sure to verify that the timer is actice and that the service runs successfully:
```bash
sudo systemctl status generate-index.timer
sudo systemctl status generate-index.service
```
## Configuring `nginx`
Separating the server block file for the `webgen` service rather than modifying the `nginx.conf` file directly makes it easier to manage configurations.

- You will need to create the direcory for server block configurations if they don't exist:
```bash
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled
```

- Move the configuration files to their respective directories:
```bash
sudo mv nginx.conf /etc/nginx/
sudo mv webgen /etc/nginx/sites-available
```

- The updated webgen server block adds a configuration to serve files located in the `/var/lib/webgen/documents` directory
**Key changes:**
1. `alias /var/lib/webgen/documents;`: Specifies the actual directory on the server to serve files for requests under `/documents`
2. `autoindex on;`: Enables directory listing. If no index.html or similar file exists in the directory, nginx will display a list of files
3. `autoindex_exact_size off;`: Displays file sizes in a human-readable format
4. `autoindex_format html;`: Formats the directory listing as an HTML page for better readability

- You can now enable the site by creating a symlink:
```bash
sudo ln -s /etc/nginx/sites-available/webgen /etc/nginx/sites-enabled/
```
- Once that's done, you will need to reload `nginx`:
```bash
sudo systemctl reload nginx
```
- You can also check the status of `nginx` to make sure it is running as expected:
```bash
sudo systemctl status nginx
```
- You can also test the `nginx` configuration by using the following:
```bash
sudo nginx -t
```

## Configuring UFW
Make sure to install `ufw` or update your server if you haven't recently:
```bash
sudo pacman -S ufw
```
- Now you can configure `ufw` using the following commands: 
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw limit shh
sudo systemctl enable --now ufw.service
```
- Once you have done that you can enable `ufw`:
```bash
sudo ufw enable
```
- To check the status you can use:
```bash
sudo ufw status verbose
```