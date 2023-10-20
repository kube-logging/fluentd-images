ARG BUILD_DEPS=" \
      make gcc g++ libc-dev \
      wget bzip2 zlib-dev git linux-headers \
      automake autoconf libtool build-base \
      ruby-dev libc6-compat geoip-dev \
      snappy-dev \
      "
# Base image based on https://gallery.ecr.aws/sumologic/kubernetes-fluentd
FROM public.ecr.aws/sumologic/kubernetes-fluentd:latest-alpine@sha256:98a9105af5199295f812020521f8ae86be927a4adb3ddbb9cf44211dd575a736 AS base
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
 # The only required fluentd dependency is the label router
 && gem install specific_install -v 0.3.8 \
 && gem specific_install -l https://github.com/kube-logging/fluent-plugin-label-router.git --ref 2ff43789f895735b7852e0a1a809280e22d9e8ef \
 && apk del .build-deps $BUILD_DEPS \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem /usr/lib/ruby/gems/3.*/gems/fluentd-*/test
 

RUN mkdir -p /buffers && chown -R fluent /buffers

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
 && apk add $BUILD_DEPS \
 && gem specific_install --debug --backtrace --verbose https://github.com/ManageIQ/kubeclient --ref 054bff2c5e31a555004be2b3c4d32fb9dc5e6a0f \
 && gem specific_install --debug --backtrace --verbose -l https://github.com/banzaicloud/fluent-plugin-gcs.git --ref 9fbff74734ddce289f9a10734a338b3c7f0065b5 \
 && gem specific_install --debug --backtrace --verbose -l https://github.com/acquia/fluent-plugin-syslog_rfc5424.git --ref 5199be67d1a385f529fa1d6b6023e95ba7fac27d \
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
 && apk add $BUILD_DEPS \
 && echo 'gem: --no-document' >> /etc/gemrc \
 && gem install specific_install -v 0.3.8 \
 && gem install bundler && bundle install \ 
 && apk del .build-deps $BUILD_DEPS \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem /usr/lib/ruby/gems/3.*/gems/fluentd-*/test


USER fluent

ENTRYPOINT ["tini",  "--", "/bin/entrypoint.sh"]
CMD ["fluentd"]