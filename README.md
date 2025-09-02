# Infrastructure

Memory before creating a VM for Docker (`free -h`):

```
              total        used        free      shared  buff/cache   available
Mem:          7.6Gi       3.3Gi       1.4Gi       2.0Mi       2.9Gi       4.0Gi
Swap:            0B          0B          0B
```

Memory after creating another VM with 2 GB RAM:

```
              total        used        free      shared  buff/cache   available
Mem:          7.6Gi       4.7Gi       606Mi       2.0Mi       2.4Gi       2.7Gi
Swap:            0B          0B          0B
```

Memory after create another VM with 3 GB RAM. Also, the host is now 24.04.1:

```
               total        used        free      shared  buff/cache   available
Mem:           7.6Gi       6.4Gi       482Mi       1.9Mi       1.0Gi       1.2Gi
Swap:             0B          0B          0B
```

## Create a VM

1. Created a  VMfrom 24.04.3. Set macvtap device to: `enp0s31f6`. This appears to be a magic number I got from somewhere and use forever.
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
* `config/deploy.yml`. Deploy a recent version of Postgres if a database is needed.
* `.kamal/secrets` and maybe other secrets stuff (e.g. `.env`). Secrets include the image registry API key.
* A registry to store the image.
* A heartbeat/up page.

I did some work with Kamal 1 to deploy Rails apps and the static site. I also did work to containerize the LimeSurvey app.

The Rails apps also need a lot of clean-up and updating. For example:

* The asset pipeline is Sprockets.
* Gems are pinned to really old versions.
* Bootstrap V4.

### Rails Apps

- [ ] `acedashboards.com`.
  - [ ] Rails 8.
  - [ ] `Dockerfile`.
  - [ ] `config/deploy.yml`.
  - [ ] `.kamal/secrets`.
  - [ ] `up` page.
- [ ] `outages.weenhanceit.com`
  - [ ] Rails 8.
  - [ ] `Dockerfile`.
  - [ ] `config/deploy.yml`.
  - [ ] `.kamal/secrets`.
  - [ ] `up` page.
- [ ] `plazachapina.ca`
  - [ ] Rails 8.
  - [ ] `Dockerfile`.
  - [ ] `config/deploy.yml`.
  - [ ] `.kamal/secrets`.
  - [ ] `up` page.

### Static Site

The static site is an off-the-shelf nginx image with the site added to its `html` directory.

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
