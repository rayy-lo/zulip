FROM debian:10

ARG DEBIAN_MIRROR

# Basic packages and dependencies of docker-systemctl-replacement
RUN echo locales locales/default_environment_locale select C.UTF-8 | debconf-set-selections \
    && echo locales locales/locales_to_be_generated select "C.UTF-8 UTF-8" | debconf-set-selections \
    && { [ ! "$DEBIAN_MIRROR" ] || sed -i "s|http://\(\w*\.\)*\.debian\.org/debian |$DEBIAN_MIRROR |" /etc/apt/sources.list; } \
    && apt-get update \
    && apt-get install --no-install-recommends -y \
           ca-certificates \
           curl \
           locales \
           openssh-server \
           python3 \
           sudo \
           systemd \
    && rm -rf /var/lib/apt/lists/*

ARG VAGRANT_UID

RUN \
    # We use https://github.com/gdraheim/docker-systemctl-replacement
    # to make services we install like PostgreSQL, Redis, etc. normally
    # managed by systemd start within Docker, which breaks normal
    # operation of systemd.
    dpkg-divert --add --rename /bin/systemctl \
    && curl -fLsS -o /bin/systemctl 'https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/v1.5.4505/files/docker/systemctl3.py' \
    && echo '93006382a98aadfd2490e521824fc870759732ff80cd012ce0dfc70d4225c803  /bin/systemctl' | sha256sum -c \
    && chmod +x /bin/systemctl \
    && ln -nsf /bin/true /usr/sbin/policy-rc.d \
    && mkdir -p /run/sshd \
    # docker-systemctl-replacement doesn’t work with template units yet:
    # https://github.com/gdraheim/docker-systemctl-replacement/issues/62
    && ln -ns /lib/systemd/system/postgresql@.service /etc/systemd/system/multi-user.target.wants/postgresql@11-main.service \
    # redis fails to start with the default configuration if IPv6 is disabled:
    # https://github.com/antirez/redis/pull/5598
    && dpkg-divert --add --rename /etc/default/redis-server \
    && printf 'ULIMIT=65536\nDAEMON_ARGS="/etc/redis/redis.conf --bind 127.0.0.1"\n' > /etc/default/redis-server \
    && mkdir /etc/systemd/system/redis-server.service.d \
    && printf '[Service]\nExecStart=\nExecStart=/usr/bin/redis-server /etc/redis/redis.conf --bind 127.0.0.1\n' > /etc/systemd/system/redis-server.service.d/override.conf \
    # Set up the vagrant user and its SSH key (globally public)
    && useradd -ms /bin/bash -u "$VAGRANT_UID" vagrant \
    && mkdir -m 700 ~vagrant/.ssh \
    && curl -fLsS -o ~vagrant/.ssh/authorized_keys 'https://raw.githubusercontent.com/hashicorp/vagrant/be7876d83644aa6bdf7f951592fdc681506bcbe6/keys/vagrant.pub' \
    && chown -R vagrant: ~vagrant/.ssh \
    && echo 'vagrant ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/vagrant

CMD ["/bin/systemctl"]

EXPOSE 22
EXPOSE 9991
