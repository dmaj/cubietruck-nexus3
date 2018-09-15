FROM alpine:3.8

# Environment
ENV     JAVA_VERSION_MAJOR=8                    \
        JAVA_VERSION_MINOR=181                  \
        JAVA_VERSION_BUILD=13                   \
        JAVA_PACKAGE=jdk                                \
        JAVA_HOME=/opt/jdk                              \
        PATH=${PATH}:/opt/jdk/bin               \
        GLIBC_VERSION=2.23-r3                   \
        ENCODING=UTF-8                                  \
        LANG=C.${ENCODING}

# Install glibc
RUN     apk upgrade --update    && \
        apk add libstdc++ curl bash ca-certificates     && \
        curl -sSL https://github.com/chrisanthropic/docker-alpine-rpi-glibc-builder/releases/download/0.0.1/glibc-${GLIBC_VERSION}.apk -o /tmp/glibc-${GLIBC_VERSION}.apk       && \
        curl -sSL https://github.com/chrisanthropic/docker-alpine-rpi-glibc-builder/releases/download/0.0.1/glibc-bin-${GLIBC_VERSION}.apk -o /tmp/glibc-bin-${GLIBC_VERSION}.apk       && \
        curl -sSL https://github.com/chrisanthropic/docker-alpine-rpi-glibc-builder/releases/download/0.0.1/glibc-i18n-${GLIBC_VERSION}.apk -o /tmp/glibc-i18n-${GLIBC_VERSION}.apk     && \
        apk add --allow-untrusted /tmp/*.apk    && \
        rm -v /tmp/*.apk        && \
        /usr/glibc-compat/bin/localedef -i en_US -f ${ENCODING} ${LANG} && \
        echo "export LANG=${LANG}" > /etc/profile.d/locale.sh   && \
        /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib      && \
        mkdir /opt

# Install Java

ADD     opt/jdk /opt/jdk

RUN     ln -s /opt/jdk1.${JAVA_VERSION_MAJOR}.0_${JAVA_VERSION_MINOR} /opt/jdk  && \
        sed -i s/#networkaddress.cache.ttl=-1/networkaddress.cache.ttl=30/ $JAVA_HOME/jre/lib/security/java.security    && \
        apk del curl glibc-i18n && \
        rm -rf  /tmp/*  \
        /var/cache/apk/*        && \
        echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf
