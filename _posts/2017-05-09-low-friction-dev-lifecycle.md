# Requirements

- New dev (or your new laptop) up and running in minutes

More specifically:

- Disposable
- One command to start
- Simple API/Architecture
- Deterministic
- All dependencies provided
- As close as possible to prod
- Fixture data that allows testing all scenarios

# PD Composer

https://github.com/ps-dev/pd-composer

- Docker Compose
- Mocks (for 3rd party dependencies)
- Proxy
- Orchestration scripts
- Ad-Hocs

# Docker Compose

- Postgres
- Cassandra
- Rabbit
- Queue Manager
- Redis
- Identity Lite
- Mocks
- Our Listeners
- Our APIs

https://github.com/ps-dev/pd-composer/blob/master/docker-compose.yml

# Mocks

Decision: Use the real services or provide mocks.

Real services:

- run everything together and get a more thorough integration
- harder

Mock services

- more easily provide various use cases with little setup
- mocks must stay in sync with the real services

Either way, you need to provide a way to bring all these up.

My team prefers to use real services for services that we develop and make
mocks for anything made by someone else (either in the company or 3rd party).

# Mocks: Our Solution

- A single express app
- Run with docker
- Mapped volumes + nodemon for easy tweaking

# Proxy

Want consistent URLs with prod/stage:

- Production - app.pluralsight.com
- Stage - app-stage.pluralsight.com
- Dev - app-dev.pluralsight.com

Provide SSL in one place.

Easy to configure if you want to test something.

# Proxy: Our Solution

Custom express app. Why?

- Simple config file
- Use tech my team is already good at

# Orchestration Scripts

Other things needed to ensure env is up to date:

- `git pull` all repos
- Get latest docker images
- Destroy everything! (constantly prove that it is disposable)
- `docker-compose up -d`
- Wait for resources to be ready
- Create database schemas
- Insert data into databases

# Ad-Hocs

Some of the steps in starting up require installing many dependencies.

Example: After creating cassandra database, we need our schema applied. We
apply that with Salt. So we have to install and configure Salt with the correct
states and pillars and install all dependencies that it has, such as a cassandra
driver.

To support a simple, one click setup, we encapsulate all of this in a docker
image. We ended up just creating a single docker image, which we refer to as
ad-hocs, that has any stuff like this installed. Our setup script delegates
any complex stuff like this to the docker image.

# Bootstrapping externally mastered data

Externally mastered data: use messages!

This is how prod gets this data and it is convenient to do in dev too (since we
  are already running our listeners!).

https://github.com/ps-dev/pd-composer/blob/master/ad-hocs/entrypoint.sh#L31

# Bootstrapping internally mastered data

SQL scripts so far.

https://github.com/ps-dev/pd-composer/blob/master/ad-hocs/entrypoint.sh#L16

# Summary

- Every day, start by running `refresh.sh` which destroys and re-creates
everything.
- Takes ~ 1 minute
- New dev: same process, but it has to download images so it takes longer (10 mins?)
