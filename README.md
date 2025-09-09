# Infrastructure

Memory on `hunapu` before creating a VM for Docker (`free -h`):

```
              total        used        free      shared  buff/cache   available
Mem:          7.6Gi       3.3Gi       1.4Gi       2.0Mi       2.9Gi       4.0Gi
Swap:            0B          0B          0B
```

Memory after creating another VM with 3 GB RAM:

```
               total        used        free      shared  buff/cache   available
Mem:           7.6Gi       6.9Gi       162Mi       1.9Mi       830Mi       711Mi
Swap:             0B          0B          0B
```

Memory on `a-22-04` with three Rails apps running (`free -h`):

```
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       527Mi        94Mi       7.0Mi       1.3Gi       1.2Gi
Swap:          1.8Gi       4.0Mi       1.8Gi
```

Memory on `postgres-a`:

```
              total        used        free      shared  buff/cache   available
Mem:          964Mi       185Mi       110Mi        57Mi       668Mi       550Mi
Swap:         1.9Gi        41Mi       1.9Gi
```

Memory on `kamal-a` with three Rails apps and LimeSurvey (with Apache) running:

```
               total        used        free      shared  buff/cache   available
Mem:           2.8Gi       1.0Gi       277Mi        72Mi       1.9Gi       1.8Gi
Swap:          2.8Gi       185Mi       2.7Gi
```

## Create a VM

1. Created a  VM from 24.04.3. Set macvtap device to: `enp0s31f6`. This appears to be a magic number I got from somewhere and use forever.
2. I asked for 3 GB of RAM.
3. I asked for 25 GB of disk. When creating the disk, _don't_ use the default LVM. It just wastes half the space. 
1. _DON'T_ install Docker as part of the installation. It will use Snap, which will lead you to no end of grief. Don't _ever_ install anything with Snap!

## Install Docker

1. Google for the [instructions to install Docker](https://docs.docker.com/engine/install/ubuntu/) from Docker `apt` repos.
1. Do the steps [here](https://docs.docker.com/engine/install/linux-postinstall/).
1. `sudo groupadd docker`.
1. `sudo usermod -aG docker $USER`.
1. Log out and back in. Or `newgrp docker`.
1. `docker run hello-world`.

## Create a Deploy User

1. Create a user `deploy` on the server.
2. Add it to the `docker` group.
3. Give it `authorized_keys` for the people who are going to deploy.

```
sudo adduser --gecos "" deploy
sudo adduser deploy docker
sudo mkdir -m 700 ~deploy/.ssh
curl https://github.com/lcreid.keys | sudo tee -a ~deploy/.ssh/authorized_keys
sudo chmod 600 ~deploy/.ssh/authorized_keys
sudo chown -r deploy:deploy ~deploy/.ssh
```

## Install Kamal on Laptop

[Notes from original install:]

I want this everywhere. Do I install it as `root`? Probably.

```bash
gem install kamal
```

If installing like the above, you need to add to your `PATH`. The executable is in `~/.gem/bin/`.

## Approach

I have three Rails apps, one static site, and one PHP app (LimeSurvey).

They all need:

* `Dockerfile`. This has to be for a recent version of Ruby, in the case of Rails apps. And Rails 8, because that's what's the latest.
* `config/deploy.yml`. Deploy a recent version of Postgres if a database is needed. See below. 
* `.kamal/secrets` and maybe other secrets stuff (e.g. `.env`). Secrets include the image registry API key.
* A registry to store the image.
* A heartbeat/up page.
* The non-static sites also need their data copied over. I think this was going to be a bit challenge, or at least it looked like it when I was trying to make a containerized version of LimeSurvey.

I did some work with Kamal 1 to deploy Rails apps and the static site. I also did work to containerize the LimeSurvey app.

The Rails apps also need a lot of clean-up and updating. For example:

* The asset pipeline is Sprockets.
* Gems are pinned to really old versions.
* Bootstrap V4.
* Use port 80 in both `Dockerfile` and `config/deploy.yml` (default in the latter).
* Remove Capistrano.

### `deploy.yml`

* The database host that the app uses (e.g. `database.yml`) must be the `<service-name>-<accessory-name>`.
* Use `user: deploy`.
* Put the database in a directory:
  ```
  directories:
    - postgres:/var/lib/postgresql/data
  ```

### Copying Data

1. Turn off the front end on the old platform.
2. Copy _each_ database with `pg_dump`, not `pg_dumpall`. This won't copy the users.
4. Deploy the app via Kamal. This should create the database and user.
5. Set the app to maintenance mode as quickly as possible???
3. Copy the dumps to somewhere accessible. Since I'm using a directory on the server for each app, I should be able to copy the dump to `kamal-a` after bringing the apps up once. The data will be accessible to the Postgres container.
6. Load the data. Can I just use `exec` on the accessory and upload the file from the laptop? I won't have `psql` on the server itself. So whatever I do will have to be either in the accessory or the app. The Docker Hub page for Postgres says you can run `psql` in the container with: `docker run -it --rm --network some-network postgres psql -h some-postgres -U postgres`.

### Rails Apps

- [x] `acedashboards.com`.
  - [x] Rails 8.
  - [x] `Dockerfile`.
  - [x] `.dockerignore` -- minimize size of image and don't leak secrets.
  - [x] `config/deploy.yml`.
  - [x] `.kamal/secrets`.
  - [x] `up` page.
    - [x] Route.
    - [x] Controller.
- [ ] `outages.weenhanceit.com`
  - [x] Rails 8.
  - [x] `Dockerfile`. -- I think Outages might have the best/most like stock Rails 8 `Dockerfile`. Better than poppies or dashboards.
  - [x] `.dockerignore` -- minimize size of image and don't leak secrets.
  - [x] `config/deploy.yml`.
  - [x] `.kamal/secrets`.
  - [x] `up` page.
    - [x] Route.
    - [x] Controller.
- [x] `plazachapina.ca`
  - [x] Rails 8.
  - [x] `Dockerfile`.
  - [x] `.dockerignore` -- minimize size of image and don't leak secrets.
  - [x] `config/deploy.yml`.
  - [x] `.kamal/secrets`.
  - [x] `bin/docker-entrypoint`.
  - [x] `up` page.
    - [x] Route.
    - [x] Controller.

Migration:

1. Shut down services.
2. Copy databases. (Practice this first.) On `postgres-a`: `sudo -u postgres pg_dump -d chapina --clean --if-exists >chapina.sql`.
3. Point router at `kamal-a`.
  * Reserve IP.
  * Forward HTTP and HTTPS to the IP -- _Don't forget to apply/save!_
5. `dotenv kamal setup` once to set up the proxy and accessories.
6. Deploy `acedashboards.com`.
7. `kamal app maintenance`.
8. Rsync database copy to database directory. Something like `~deploy/acedashboards/postgres`. On `postgres-a`: `rsync dashboard_prod.sql kamal-a:`
9. On `kamal-a`: `sudo mv outages_prod.sql /home/deploy/outages-db/postgres`.
10. `sudo chown 70:70 /home/deploy/outages-db/postgres/outages_prod.sql`.
10. Log in to the accessory `dotenv kamal accessory exec -i db /bin/bash`, and type: `psql -U acedashboards -d dashboard_prod -f /var/lib/postgresql/data/dashboard_prod.sql -h acedashboards-db`. `psql -U outages -h outages-db -d outages_prod -f /var/lib/postgresql/data/outages_prod.sql`.
10. `kamal app live`.

### Static Site

The static site is an off-the-shelf nginx image with the site added to its `html` directory.

Holy crap! This was super-easy. But I can't deploy from the repo, because I deploy `_site`. So: `context: .`.

### LimeSurvey/PHP App

I found [this](https://github.com/martialblog/docker-limesurvey).

- [ ] `limesurvey.verapax.org`
  - [x] `kamal init`.
  - [x] `Dockerfile`. Took the `Dockerfile` from the above link. Version 6.0 apache. 
  - [ ] `.dockerignore` -- minimize size of image and don't leak secrets.
  - [x] `config/deploy.yml`.
  - [x] `.kamal/secrets`.
  - [x] `bin/docker-entrypoint`. Not in bin, but is part of the `Dockerfile`.
  - [x] Doh! Still need `up` page. Just configure it to hit the home page, even though that triggers a bunch of activity.

When creating the database, or really the whole image, I have to force it _not_ to use a prefix. The original database didn't use a prefix so the restore isn't going to work.

Oh, this makes me think I had this problem before: Forcing TLS/SSL is set in the LimeSurvey database. So when you load the database dump in a local docker compose environment, you get grief because you don't have a certificate. And now that it's up, also grief because the database doesn't 

```
$ docker compose exec -i limesurvey-db /bin/bash
root@7dbc06aaa178:/# psql -U limesurvey
psql (17.6 (Debian 17.6-1.pgdg13+1))
Type "help" for help.

limesurvey=# select * from settings_global where stg_name like '%ssl%';
   stg_name   | stg_value 
--------------+-----------
 force_ssl    | on
 emailsmtpssl | ssl
(2 rows)

limesurvey=# update settings_global set stg_value = 'off' where stg_name = 'force_ssl';
UPDATE 1
limesurvey=# select * from settings_global where stg_name like '%ssl%';
   stg_name   | stg_value 
--------------+-----------
 emailsmtpssl | ssl
 force_ssl    | off
(2 rows)
```

Limesurvey needs e-mail for password resets, etc. (Probably other apps do, too.) Needed to get an app password from Gmail.

### Backups

I really only need backups now of the databases. If something goes bad on the server, I just deploy the server. If we had uploads, I'd have to back them up too, but that's not the case so far (will be for LimeSurvey). Previously I backed up the VM, and the database VM. There is a Postgres image to do backups. It might put them on S3. It runs as another accessory.

Exclude the whole server from the host backups, as I did with the other VMs, since it was backing itself up? Yes. That's what I did.

Backing up `kamal-a`. That's a lot being backed up that doesn't need to be backed up, but I can tune this, I guess. I really only need to back up the `~deploy` directory, as that's where the database data is.

## Configuring

`accessories` is how you set up things like the database server.

There's a command to test the registry set-up, and the registry set-up isn't as easy as they lead you to think: `kamal registry login`.

`config.yml` is YAML, so you have to be really careful about indentation. The example file that's generated makes it easy to uncomment without really thinking about the level of indentation.

It looks like accessories run on `roles`, and the `role`s are defined in the `servers` section. I haven't explored why you define an accessory on a `host` or `hosts`.

## Secrets

I spun my wheels a long time on secrets. In the end, I chose to use the `.env` approach. _If you use Kamal and `dotenv` correctly_, Kamal will take what's in your `.env` and arrange for it to be in a file in the host that can be used to pass the environment to the container. What no one tells you is that you have to prefix your commands with `dotenv`, like so:

```bash
dotenv kamal deploy
```

For poppies, we just kept `secrets` out of `git` and kept them there.

## Total Adventure Getting Dockerfile Right

So I took the Rails `Dockerfile` and dropped it in the ACE website app.

* The Dockerfile seems to be built for the latest way Rails deals with the front-end. So it doesn't leave `npm` and `yarn` in the image. It just uses them to build the assets and then finishes the image from an earlier layer. I had to explicitly copy the executables after they were built. [Edit.] I'm not sure what I meant by the preceding, because the build should build the assets and then we don't need `npm` or `yarn` anymore.
* The `docker-compose.yml` (maybe this can just be `compose.yml` now?) for production is quite different from my development one. The secrets that are set in `.env` do _not_ get passed automatically to the containers. You have to have entries in the `environment` thingy that explicitly set the container's environment to `docker compose`'s environment.
* This experience shows how I didn't really understand Docker and Compose when I built the development images. I would like to understand better when the container is re-created. I may be able to leave more of the gems and cached stuff in the container, rather than in the local file system. 

## Outstanding Questions

Kamal 2 questions:

* How to run Kamal logging in as a regular user and not `root`? Looks like you just install Docker by hand, which I did: https://kamal-deploy.org/docs/configuration/ssh/. Then set the SSH user in the config file.
* What's a good way to test this? I want to be able to look like I'm coming to the same IP with different URLs, like I do now with Nginx figuring out where to route the request. But I want to do this while the real sites are still live and DNS is pointing where it should. I know how to make the requests from `curl`, but I'd like to do it from the browser. Did I figure out how to do that? Looks like maybe just host file, although maybe I can make a Chrome profile to use a different host file.
