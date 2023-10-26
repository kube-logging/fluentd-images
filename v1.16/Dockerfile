FROM public.ecr.aws/sumologic/kubernetes-fluentd:latest-alpine@sha256:98a9105af5199295f812020521f8ae86be927a4adb3ddbb9cf44211dd575a736 AS sumo

# TODO copy deps from the sumologic image to avoid the 999:999 user/group id

# Adapted from https://github.com/SumoLogic/sumologic-kubernetes-fluentd/blob/main/alpine.Dockerfile#L102C1-L142C16
FROM ruby:3.1.4-alpine3.17 as base

ARG BUILD_DEPS=" \
      make gcc g++ libc-dev \
      wget bzip2 zlib-dev git linux-headers \
      automake autoconf libtool build-base \
      ruby-dev libc6-compat geoip-dev \
      snappy-dev gnupg \
      "

RUN addgroup -S -g 101 fluent && adduser -S -G fluent -u 100 fluent \
    # for log storage (maybe shared with host)
    && mkdir -p /fluentd/log \
    # configuration/plugins path (default: copied from .)
    && mkdir -p /fluentd/etc /fluentd/plugins \
    && chown -R fluent /fluentd && chgrp -R fluent /fluentd \
    && mkdir -p /buffers && chown -R fluent /buffers

COPY --from=sumo --chown=fluent:fluent /usr/local/bundle /usr/local/bundle

RUN apk add --no-cache \
        ca-certificates \
        ruby ruby-irb ruby-etc \
        tini libmaxminddb geoip \
        snappy \
 && apk add --no-cache $BUILD_DEPS \
 && echo 'gem: --no-document' >> /etc/gemrc \
 # The only required fluentd dependency is the label router
 # kubeclient install is upgrading the one from the base image
 && fluent-gem install specific_install -v 0.3.8 \
 && fluent-gem specific_install -l https://github.com/ManageIQ/kubeclient.git --ref 054bff2c5e31a555004be2b3c4d32fb9dc5e6a0f \
 && fluent-gem specific_install -l https://github.com/kube-logging/fluent-plugin-label-router.git --ref 2ff43789f895735b7852e0a1a809280e22d9e8ef \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

COPY fluent.conf /fluentd/etc/
COPY entrypoint.sh /bin/
COPY healthy.sh /bin/

ENV FLUENTD_CONF="fluent.conf"
ENV LD_PRELOAD=""
EXPOSE 24224 5140

USER fluent

### Minimal image, filters only
FROM base AS minimal

ADD ./Gemfile.filters /Gemfile.filters
ADD ./Gemfile.filters.lock /Gemfile.filters.lock

USER root

ENV BUNDLE_WITHOUT="output,test,development"

RUN apk add --no-cache $BUILD_DEPS \
 && touch /etc/gemrc \
 && fluent-gem install --file Gemfile.filters \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

USER fluent

### Full image
FROM minimal as full

ADD ./Gemfile.outputs /Gemfile.outputs
ADD ./Gemfile.outputs.lock /Gemfile.outputs.lock

USER root

RUN apk add --no-cache $BUILD_DEPS \
 && touch /etc/gemrc \
 && fluent-gem specific_install -l https://github.com/kube-logging/fluent-plugin-gcs.git --ref 71a0932e75509545a2d28e337642ee1f973cca90 \
 && fluent-gem specific_install -l https://github.com/acquia/fluent-plugin-syslog_rfc5424.git --ref 5199be67d1a385f529fa1d6b6023e95ba7fac27d \
 && fluent-gem install --file /Gemfile.outputs \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

USER fluent

ENTRYPOINT ["tini",  "--", "/bin/entrypoint.sh"]
CMD ["fluentd"]
