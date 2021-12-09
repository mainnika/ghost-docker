# syntax = docker/dockerfile:1.2
# https://docs.ghost.org/faq/node-versions/
# https://github.com/nodejs/Release (looking for "LTS")
# https://github.com/TryGhost/Ghost/blob/v4.1.2/package.json#L38
FROM registry.access.redhat.com/ubi8/ubi as gosu-bin

RUN dnf makecache

# grab gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.12
RUN set -eux; \
	dnf install -yq https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm; \
	dnf install -yq ca-certificates gnupg2 wget dpkg; \
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	\
# verify the signature
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	\
# verify that the binary works
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true

FROM registry.access.redhat.com/ubi8/ubi as final

RUN dnf makecache

RUN dnf install -yq patch
RUN dnf module install -yq nodejs:14

RUN groupadd --gid 1000 node \
  && useradd --uid 1000 --gid node --shell /bin/bash --create-home node

COPY --from=gosu-bin /usr/local/bin/gosu /usr/local/bin/gosu

ENV NODE_ENV production

ENV GHOST_CLI_VERSION 1.18.1
RUN set -eux; \
	npm install -g "ghost-cli@$GHOST_CLI_VERSION"; \
	npm cache clean --force

ENV GHOST_INSTALL /var/lib/ghost
ENV GHOST_CONTENT /var/lib/ghost/content

ENV GHOST_VERSION 4.27.1

COPY yarn-adjust-network-timeout.patch /tmp/yarn-adjust-network-timeout.patch

RUN set -eux; \
	patch -d "$(npm config get prefix)/lib" -p1 < /tmp/yarn-adjust-network-timeout.patch; \
	\
	mkdir -p "$GHOST_INSTALL"; \
	chown node:node "$GHOST_INSTALL"; \
	\
# ignore warnings about mysql setup
	gosu node ghost install "$GHOST_VERSION" --db mysql --dbhost mysql --no-prompt --no-stack --no-setup --dir "$GHOST_INSTALL" || true; \
	\
# tell Ghost to listen on all ips and not prompt for additional configuration
	cd "$GHOST_INSTALL"; \
	gosu node ghost config --no-prompt --ip 0.0.0.0 --port 2368 --url http://localhost:2368 --db mysql --dbhost mysql --dbuser ghost --dbpass ghost --dbname ghost; \
	gosu node ghost config paths.contentPath "$GHOST_CONTENT"; \
	\
# make a config.json symlink for NODE_ENV=development (and sanity check that it's correct)
	gosu node ln -s config.production.json "$GHOST_INSTALL/config.development.json"; \
	readlink -f "$GHOST_INSTALL/config.development.json"; \
	\
# need to save initial content for pre-seeding empty volumes
	mv "$GHOST_CONTENT" "$GHOST_INSTALL/content.orig"; \
	mkdir -p "$GHOST_CONTENT"; \
	chown node:node "$GHOST_CONTENT"; \
	chmod 1777 "$GHOST_CONTENT"; \
	\
	gosu node npx yarn cache clean; \
	gosu node npx npm cache clean --force; \
	dnf clean all; \
	npm cache clean --force; \
	rm -rv /tmp/yarn* /tmp/v8*

WORKDIR $GHOST_INSTALL
VOLUME $GHOST_CONTENT

COPY docker-entrypoint.sh /usr/local/bin
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 2368
CMD ["node", "current/index.js"]
