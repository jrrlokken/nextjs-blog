---
title: "MaskUp Deployment Guide"
date: "2021-03-01"
---

Prerequisites:

- $10 Digital Ocean droplet — I tried this with a $5 droplet first, but ran into ENOMEM issues
- A domain. I use [Hover.com](http://hover.com) to manage my domains
  - Add the domain to 'Domains' under you digital ocean droplet and follow the directions for setting the DNS appropriately. I setup two subdomains: [maskup.joshualokken.tech](http://maskup.joshualokken.tech) for the frontend, and [api.maskup.joshualokken.tech](http://api.maskup.joshualokken.tech) for the backend

Login to your droplet via SSH, or open the console via the Digital Ocean control panel. You can do this all as the 'root' user, or setup a user that has sudo privileges. I did the latter, but will assume the root user for this write-up.

- Install node and npm on the droplet (I used nvm (Node Version Manager), but you can use the built-in package management system on your droplet. While you're here, also install nginx as the webserver and pm2 to run them.

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
# apt update
# apt install nodejs nginx git
# npm install --global pm2
```

</div>

- Make sure your .env files have the production variables in them:

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
FRONTEND_URL='https://your_new_frontend_url' (ie. https://maskup.joshualokken.tech)
```

</div>

- Make sure your frontend/config.js has the correct production endpoint set:

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
productionEndpoint: `https://your_new_backend_url/api/graphql` (ie. https://api.maskup.joshualokken.tech/api/graphql)
```

</div>

- And that your package.json files have the appropriate scripts defined:

  - frontend/package.json:

  <div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

  ```jsx
  "scripts": {
      "dev": "next -p 7777",
      "build": "next build",
      "start": "next start -p 7777",
      "test": "NODE_ENV=test jest --watch",
  },
  ```

  </div>

  - backend/package.json:

  <div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

  ```jsx
  "scripts": {
      "dev": "keystone-next",
      "start": "keystone-next",
      "build": "keystone-next build"
  },
  ```

  </div>

- Now:

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
# cd /var/www
# git clone <your repository here>
# cd <project_name>/backend
# npm install
# npm run build
# pm2 start npm --name "backend" -- start
# cd ../frontend
# npm install
# npm run build
# pm2 start npm --name "frontend" -- start

```

</div>

- At this point, you probably want to get SSL certs for your domain(s). I setup separate certs for the frontend and backend, but you can probably get by just fine with one cert for the root domain. In short, I visited [https://certbot.eff.org/](https://certbot.eff.org/) and followed the instructions for creating certificates for my server (nginx) and platform (Debian 10 - Buster). I can write up detailed instructions on how I accomplished this if needed ;)

- Configure nginx (listens on port 80 and 443, redirects all requests to https))

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
# vim /etc/nginx/sites-available/<frontend_domain>.conf

server {
        listen 80;
        server_name maskup.joshualokken.tech;
        return 301 https://maskup.joshualokken.tech$request_uri;
}

server {
        listen 443 ssl;

        ssl_certificate /etc/letsencrypt/live/maskup.joshualokken.tech/cert.pem;
        ssl_certificate_key /etc/letsencrypt/live/maskup.joshualokken.tech/privkey.pem;

        server_name maskup.joshualokken.tech;
        location / {
                proxy_pass http://localhost:7777;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
```

```jsx
# vim /etc/nginx/sites-available/<backend_domain>.conf
# ln -s /etc/nginx/sites-available/<backend_domain>.conf /etc/nginx/sites-enabled/

server {
        listen 80;
        server_name api.maskup.joshulokken.tech;
        return 301 https://api.maskup.joshualokken.tech;
}

server {
        listen 443 ssl;

        ssl_certificate /etc/letsencrypt/live/api.maskup.joshualokken.tech/cert.pem;
        ssl_certificate_key /etc/letsencrypt/live/api.maskup.joshualokken.tech/privkey.pem;

        server_name api.maskup.joshualokken.tech;
        location / {
                proxy_pass http://localhost:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
```

```jsx
# ln -s /etc/nginx/sites-available/<frontend_domain>.conf /etc/nginx/sites-enabled/
# ln -s /etc/nginx/sites-available/<backend_domain>.conf /etc/nginx/sites-enabled/
# rm /etc/nginx/sites-enabled/default
# sudo systemctl start nginx
# pm2 restart all
```

</div>

- To check the status of your apps:

<div style="background-color: rgb(50, 50, 50); color: white; padding: 0 5px;">

```jsx
# pm2 ls
┌─────┬────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name               │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0   │ maskup-backend     │ default     │ 0.37.2  │ fork    │ 23801    │ 42h    │ 4    │ online    │ 0%       │ 51.8mb   │ joshua   │ disabled │
│ 1   │ maskup-frontend    │ default     │ 0.37.2  │ fork    │ 12511    │ 20h    │ 84   │ online    │ 0%       │ 55.5mb   │ joshua   │ disabled │
└─────┴────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

# netstat -tupln | grep LISTEN

(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:7777            0.0.0.0:*               LISTEN      12559/node
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
tcp6       0      0 :::3000                 :::*                    LISTEN      23850/
```

</div>

This is the down-and-dirty — I'm sure there are things I've left out, and I assumed some things along the way (like you know how to use the 'vim' editor, etc.). If you have _any_ questions, please do not hesitate to hit me up in Slack. Cheers 🙂
