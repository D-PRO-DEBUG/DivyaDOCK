FROM node:22-bookworm AS builder
RUN apt-get update && apt-get install -y jq
# global installs need root permissions, so have to happen before we switch to
# the node user
RUN npm i -g pnpm@10
# node images create a non-root user that we can use
USER node
WORKDIR /home/node/build

COPY --chown=node:node *.* .
COPY --chown=node:node api/ api/
COPY --chown=node:node shared/ shared/
COPY --chown=node:node tools/ tools/
COPY --chown=node:node curriculum/ curriculum/
# TODO: AFAIK it's just the intro translations. Those should be folded into the
# curriculum and then we can remove this.
COPY --chown=node:node client/ client/

RUN pnpm config set dedupe-peer-dependents false
# While we want to ignore scripts generally, we do need to generate the prisma
# client. Note: npx is the simplest way to generate the client without us having
# to have prisma as a prod dependency. The jq tricks are to ensure we're using
# the right version of prisma.
RUN pnpm install -F=api -F=curriculum -F tools/scripts/build -F challenge-parser \
--frozen-lockfile --ignore-scripts
RUN cd api && npx prisma@$(jq -r '.devDependencies.prisma' < package.json) generate

# The api needs to source curriculum.json and build:curriculum relies on the
