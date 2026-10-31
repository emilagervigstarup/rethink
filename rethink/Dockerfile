# BUILD_FROM must be declared before the first FROM — args declared between stages
# belong to the preceding stage and are invisible to FROM instructions.
ARG BUILD_FROM

# Build stage: compile rethink from the pinned fork revision
FROM alpine:3.21 AS build

RUN apk add --no-cache nodejs npm git

# jmcjm/rethink `deploy` branch: upstream master + RH90V9_WW dryer handler
# + Y_V8_Y___W.B32QEUK EU washer extensions. Bump on every add-on update.
ARG RETHINK_REPO=https://github.com/jmcjm/rethink.git
ARG RETHINK_REV=c16217ea4da6b6ae87e9b116c283d854413d4be0

RUN git clone "$RETHINK_REPO" /src \
    && git -C /src checkout "$RETHINK_REV"

WORKDIR /src
RUN npm ci --ignore-scripts \
    && npm run build \
    && npm prune --omit=dev

# Runtime stage: Home Assistant base image (s6 entrypoint disabled via init: false)
FROM $BUILD_FROM

# ca-certificates is upgraded to drop the compromised eMudhra root, same as upstream's image.
# openssl is gone: rethink now handles certificates in-process.
RUN apk upgrade --no-cache \
    && apk add --no-cache nodejs ca-certificates jq

WORKDIR /app
COPY --from=build /src/package.json /src/package-lock.json ./
COPY --from=build /src/node_modules ./node_modules
COPY --from=build /src/dist ./dist
COPY run.sh /app/run.sh
RUN chmod +x /app/run.sh

ENTRYPOINT []
CMD ["/app/run.sh"]
