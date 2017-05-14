# Requirements

- fixture data that allows testing all scenarios
- default configurations
- all dependencies provided (e.g. don't have to install Postgres manually)
- as close to single command startup as possible
- proxy (to put everything on one domain like prod)
- everything easy to configure
- easy to develop the service you're actually working on (prefer local)
- mocks are easy to modify to create specific scenarios

# Dependencies on services

Decision: Use the real services or provide mocks.

Real services:

- run everything together and get a more thorough integration

Mock services

- more easily provide various use cases with little setup

Either way, you need to provide a way to bring all these up.

My team prefers to use real services for services that we develop and make
mocks for anything made by someone else (either in the company or 3rd party).

# Backend services
