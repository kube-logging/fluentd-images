ARG BUILD_DEPS=" \
      make gcc g++ libc-dev \
      wget bzip2 zlib-dev git linux-headers \
      automake autoconf libtool build-base \
      ruby-dev libc6-compat geoip-dev \
      snappy-dev \
      "
### Very base image
FROM alpine:3.18 AS base
ENV FLUENTD_VERSION 1.16.2

ADD ./Gemfile /Gemfile

ARG BUILD_DEPS
# Do not split this into multiple RUN!
# Docker creates a layer for every RUN-Statement
# therefore an 'apk delete' has no effect
RUN apk update \
 && apk add --no-cache \
        ca-certificates \
        ruby ruby-irb ruby-etc \
        tini libmaxminddb geoip \
        snappy \
 && apk add --no-cache --virtual .build-deps \
        build-base \
        ruby-dev gnupg \
 && apk add $BUILD_DEPS \
 && echo 'gem: --no-document' >> /etc/gemrc \
 && gem install fluentd -v ${FLUENTD_VERSION} \
 # The only required fluentd dependency is the label router
 && gem install specific_install -v 0.3.8 \
 && gem specific_install -l https://github.com/kube-logging/fluent-plugin-label-router.git --ref 2ff43789f895735b7852e0a1a809280e22d9e8ef \
 && apk del .build-deps $BUILD_DEPS \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem /usr/lib/ruby/gems/3.*/gems/fluentd-*/test
 

RUN addgroup -S fluent && adduser -S -G fluent fluent \
    # for log storage (maybe shared with host)
    && mkdir -p /fluentd/log \
    # configuration/plugins path (default: copied from .)
    && mkdir -p /fluentd/etc /fluentd/plugins \
    && chown -R fluent /fluentd && chgrp -R fluent /fluentd \
    && mkdir -p /buffers && chown -R fluent /buffers

COPY fluent.conf /fluentd/etc/
COPY entrypoint.sh /bin/
COPY healthy.sh /bin/


ENV FLUENTD_CONF="fluent.conf"

ENV LD_PRELOAD=""
EXPOSE 24224 5140
USER fluent
 
### Minimal image
FROM base AS minimal
ADD ./Gemfile /Gemfile
ADD ./Gemfile.lock /Gemfile.lock
ARG BUILD_DEPS
USER root
# Do not split this into multiple RUN!
# Docker creates a layer for every RUN-Statement
# therefore an 'apk delete' has no effect
RUN apk update \
 && apk add --no-cache \
        ca-certificates \
        ruby ruby-irb ruby-etc \
        tini libmaxminddb geoip \
        snappy \
 && apk add --no-cache --virtual .build-deps \
        build-base \
        ruby-dev gnupg \
 && apk add $BUILD_DEPS \
 && gem specific_install -l https://github.com/SumoLogic/sumologic-kubernetes-fluentd.git --ref 0bf48d864491de5f167fe9ca8bfe70afcc54535f -d fluent-plugin-kubernetes-metadata-filter \
 && gem specific_install -l https://github.com/SumoLogic/sumologic-kubernetes-fluentd.git --ref 0bf48d864491de5f167fe9ca8bfe70afcc54535f -d fluent-plugin-enhance-k8s-metadata \
 && gem specific_install -l https://github.com/SumoLogic/sumologic-kubernetes-fluentd.git --ref 0bf48d864491de5f167fe9ca8bfe70afcc54535f -d fluent-plugin-kubernetes-sumologic \
 && gem specific_install https://github.com/abonas/kubeclient --ref 6b40464cd71e2789ea6de94f618f793a8db9ac49 \
 && gem specific_install -l https://github.com/banzaicloud/fluent-plugin-gcs.git --ref 9fbff74734ddce289f9a10734a338b3c7f0065b5 \
 && gem specific_install -l https://github.com/acquia/fluent-plugin-syslog_rfc5424.git --ref 5199be67d1a385f529fa1d6b6023e95ba7fac27d \
 && apk del .build-deps $BUILD_DEPS \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem /usr/lib/ruby/gems/3.*/gems/fluentd-*/test

USER fluent

### Full image
FROM minimal as full
ARG BUILD_DEPS
USER root
# Do not split this into multiple RUN!
# Docker creates a layer for every RUN-Statement
# therefore an 'apk delete' has no effect
RUN apk update \
 && apk add --no-cache \
        ca-certificates \
        ruby ruby-irb ruby-etc \
        tini libmaxminddb geoip \
        snappy \
 && apk add --no-cache --virtual .build-deps \
        build-base \
        ruby-dev gnupg \
 && apk add $BUILD_DEPS \
 && echo 'gem: --no-document' >> /etc/gemrc \
 # git only deps first as they would be installed to a different location if used without specific_install
 && gem install specific_install -v 0.3.8 \
 && gem install bundler && bundle install \ 
 && apk del .build-deps $BUILD_DEPS \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem /usr/lib/ruby/gems/3.*/gems/fluentd-*/test


USER fluent

ENTRYPOINT ["tini",  "--", "/bin/entrypoint.sh"]
CMD ["fluentd"]