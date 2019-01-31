# Tutorial for Discourse Image Building / Customization

## Create Discourse App Docker Image

We will "mis-use" the launcher provided by discourse_docker to create the docker image we want for the discourse web server.

1. Clone from [https://github.com/discourse/discourse_docker](https://github.com/discourse/discourse_docker) to local environment
2. Setup temp Redis and Postgres in local environment
    1. `docker run --name redis -p 6379:6379 -d redis redis-server --protected-mode no`
    2. `docker run --name psql -p 5432:5432 -e POSTGRES_PASSWORD=JackStraw123 -d postgres`
3. Create `containers/web_only.yml` as shown below
    1. The env var is not relevant to k8s, just for building the local image, fill in something works for your local environment
    2. Determine the plugins you want to install with your discourse setting here
4. Tips: in case you download redis, it might be in protected mode and doesn't allow docker guest to host connection, start it with `redis-server --protected-mode no`
5. Create the docker images and upload the images to your k8s docker registry. Let's say we are using Google Cloud Registry:
    1. Create docker image by discourse's launcher: `./launcher bootstrap web_only`
    2. Verify created: `docker images`
    3. Upload image to registry with the commands below.

```
docker tag local_discourse/web_only gcr.io/**my-cluster**/discourse:latest
gcloud docker -- push gcr.io/**my-cluster**/discourse:latest
```
web_only.yml here

```
templates:
  - "templates/web.template.yml"
  - "templates/web.ratelimited.template.yml"

links:
  - link:
      name: redis
      alias: redis
  - link:
      name: psql
      alias: psql  

env:
  LANG: en_US.UTF-8
  UNICORN_WORKERS: 2
  DISCOURSE_DB_USERNAME: ''
  DISCOURSE_DB_PASSWORD: ''
  DISCOURSE_DB_HOST: ''
  DISCOURSE_DB_NAME: ''
  DISCOURSE_DEVELOPER_EMAILS: ''
  DISCOURSE_HOSTNAME: 'localhost'
  DISCOURSE_REDIS_HOST: ''

hooks:
  after_code:
    - exec:
        cd: $home/plugins
        cmd:
          - mkdir -p plugins
          - git clone https://github.com/discourse/docker_manager.git
          - git clone https://github.com/discourse/discourse-solved.git
          - git clone https://github.com/discourse/discourse-voting.git
          - git clone https://github.com/discourse/discourse-slack-official.git
          - git clone https://github.com/discourse/discourse-assign.git
run:
  - exec:
      cd: /var/www/discourse
      cmd:
        - sed -i 's/GlobalSetting.serve_static_assets/true/' config/environments/production.rb
        - bash -c "touch -a /shared/log/rails/{sidekiq,puma.err,puma}.log"
        - bash -c "ln -s /shared/log/rails/{sidekiq,puma.err,puma}.log log/"
        - sed -i 's/default \$scheme;/default https;/' /etc/nginx/conf.d/discourse.conf
  - exec:
      cd: $home
      cmd:
        - mkdir -p public/discussion
        - cd public/discussion && ln -s ../uploads && ln -s ../backups
  - replace:
     global: true
     filename: /etc/nginx/conf.d/discourse.conf
     from: proxy_pass http://discourse;
     to: |
        rewrite ^/(.*)$ /discussion/$1 break;
        proxy_pass http://discourse;
  - replace:
     filename: /etc/nginx/conf.d/discourse.conf
     from: etag off;
     to: |
        etag off;
        location /discussion {
           rewrite ^/discussion/?(.*)$ /$1;
        }
  - replace:
       filename: /etc/nginx/conf.d/discourse.conf
       from: $proxy_add_x_forwarded_for
       to: $http_your_original_ip_header
       global: true      
```
