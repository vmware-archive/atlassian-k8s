FROM openjdk:8
# based on https://github.com/cptactionhank/docker-atlassian-bamboo

# Setup useful environment variables
ENV BAMBOO_HOME     /var/atlassian/bamboo
ENV BAMBOO_INSTALL  /opt/atlassian/bamboo
ENV BAMBOO_VERSION  6.1.0

# Install Atlassian Bamboo and helper tools and setup initial home
# directory structure.
RUN apt-get update --quiet \
    && apt-get install --quiet --yes --no-install-recommends xmlstarlet libtcnative-1 \
    && curl --silent https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | bash \
    && apt-get install --quiet --yes --no-install-recommends git-lfs \
    && git lfs install 

RUN mkdir -p                  "${BAMBOO_HOME}/lib" "${BAMBOO_INSTALL}"\
    && curl -Ls               "https://www.atlassian.com/software/bamboo/downloads/binary/atlassian-bamboo-${BAMBOO_VERSION}.tar.gz" | tar -zx --directory  "${BAMBOO_INSTALL}" --strip-components=1 --no-same-owner \
    && curl -Ls                "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.40.tar.gz" | tar -xz --directory "${BAMBOO_INSTALL}/lib" --strip-components=1 --no-same-owner "mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar" \
    && chmod -R 700           "${BAMBOO_INSTALL}" "${BAMBOO_HOME}" \
    && chown -R daemon:daemon "${BAMBOO_INSTALL}" "${BAMBOO_HOME}" \
    && sed --in-place         's/^# umask 0027$/umask 0027/g' "${BAMBOO_INSTALL}/bin/setenv.sh" \
    && xmlstarlet             ed --inplace \
        --delete              "Server/Service/Engine/Host/@xmlValidation" \
        --delete              "Server/Service/Engine/Host/@xmlNamespaceAware" \
                              "${BAMBOO_INSTALL}/conf/server.xml" \
    && touch -d "@0"          "${BAMBOO_INSTALL}/conf/server.xml"


# Add gcloud repo and tools
RUN echo "deb http://packages.cloud.google.com/apt cloud-sdk-stretch main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
RUN curl -Ls https://packages.cloud.google.com/apt/doc/apt-key.gpg -o /tmp/apt-key.gpg
RUN apt-key add /tmp/apt-key.gpg \
        && rm -f /tmp/apt-key.gpg \ 
	&& apt-get update --quiet \ 
        && apt-get install google-cloud-sdk kubectl  --quiet --yes --no-install-recommends \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/* 

# Add kubecfg binary
ADD https://github.com/ksonnet/kubecfg/releases/download/v0.5.0/kubecfg-linux-amd64 /usr/bin/kubecfg
RUN chmod +x /usr/bin/kubecfg

# Use the default unprivileged account. This could be considered bad practice
# on systems where multiple processes end up being executed by 'daemon' but
# here we only ever run one process anyway.
#USER daemon:daemon

# Expose default HTTP and SSH ports.
EXPOSE 8085 54663

# Set volume mount points for installation and home directory. Changes to the
# home directory needs to be persisted as well as parts of the installation
# directory due to eg. logs.
VOLUME ["/var/atlassian/bamboo","/opt/atlassian/bamboo/logs"]

# Set the default working directory as the Bamboo home directory.
WORKDIR /var/atlassian/bamboo

COPY "docker-entrypoint.sh" "/"
ENTRYPOINT ["/docker-entrypoint.sh"]

# Run Atlassian Bamboo as a foreground process by default.
CMD ["/opt/atlassian/bamboo/bin/start-bamboo.sh", "-fg"]
