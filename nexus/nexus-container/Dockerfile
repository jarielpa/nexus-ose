FROM centos:7

EXPOSE 8081

ENV NEXUS_VERSION 2.14.2-SNAPSHOT
ENV NEXUS_HOME /opt/nexus/nexus
ENV NEXUS_WORK /sonatype-work
ENV NEXUS_REPOS /repositories

RUN cd /var/tmp \
  && curl --fail --silent --location --retry 3 -O \
  --header "Cookie: oraclelicense=accept-securebackup-cookie; " \
  http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.rpm \
  && rpm -Ui jdk-8u101-linux-x64.rpm \
  && rm -rf jdk-8u101-linux-x64.rpm

# Install jq
RUN curl --fail --silent --location --retry 3 \
    https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64 \
    -o /usr/local/bin/jq \
    && chmod 755 /usr/local/bin/jq

RUN yum install zip unzip -y

# Install the binaries
RUN mkdir -p /opt/nexus/nexus \
  && curl  -O --fail --silent --location --retry 3 \
    https://github.com/jarielpa/nexus-public/releases/download/2.14/nexus-2.14.1-01-bundle-fix.zip \
   && unzip nexus-2.14.1-01-bundle-fix.zip -d /tmp/nexus-${NEXUS_VERSION} \
   && mv /tmp/nexus-${NEXUS_VERSION}/* /opt/nexus/nexus/ \
   && rm -rf /tmp/nexus-${NEXUS_VERSION}
#  | gunzip \
#  | tar x -C /tmp nexus-${NEXUS_VERSION} \
# && mv /tmp/nexus-${NEXUS_VERSION}/* /opt/nexus/nexus/ \
# && rm -rf /tmp/nexus-${NEXUS_VERSION}


RUN chmod 777 $NEXUS_HOME \
   && mkdir -p $NEXUS_WORK && chmod -R 777 $NEXUS_WORK \
   && mkdir -p $NEXUS_REPOS && chmod -R 777 $NEXUS_REPOS 

RUN groupadd -r jboss -g 185 && useradd -u 185 -r -g jboss -m -d /home/jboss -s /sbin/nologin -c "JBoss user" jboss

COPY scripts/readinessProbe.sh /usr/local/bin/readinessProbe.sh
COPY scripts/fixPermissions.sh /usr/local/bin/fixPermissions.sh
COPY scripts/addrepo.sh /usr/local/bin/addrepo.sh
COPY scripts/addjbossrepos.sh /usr/local/bin/addjbossrepos.sh

RUN chmod 755 /usr/local/bin/fixPermissions.sh \
  && chmod 755 /usr/local/bin/readinessProbe.sh \
  && chmod 755 /usr/local/bin/addrepo.sh \
  && chmod 755 /usr/local/bin/addjbossrepos.sh \
  && /usr/local/bin/fixPermissions.sh /opt/nexus \
  && /usr/local/bin/fixPermissions.sh $NEXUS_WORK \
  && /usr/local/bin/fixPermissions.sh $NEXUS_REPOS

RUN mkdir /conf
RUN /usr/local/bin/fixPermissions.sh /conf

# ADD nexus.xml as a workaround until post-deployment scripts work
COPY scripts/nexus.xml $NEXUS_HOME/conf/
RUN /usr/local/bin/fixPermissions.sh $NEXUS_HOME/conf

USER jboss

VOLUME ["/sonatype-work"]
VOLUME ["/repositories"]

WORKDIR $NEXUS_HOME
ENV CONTEXT_PATH /
ENV MAX_HEAP 768m
ENV MIN_HEAP 256m
ENV JAVA_OPTS -server -XX:MaxPermSize=192m -Djava.net.preferIPv4Stack=true -Dhttp.proxyHost=http://proxy.posti.fi -Dhttp.proxyPort=8086 -Dhttps.proxyHost=http://proxy.posti.fi -Dhttps.proxyPort=8086 -Dhttp.nonProxyHosts=maven.itella.com
ENV LAUNCHER_CONF ${NEXUS_HOME}/conf/jetty.xml ${NEXUS_HOME}/conf/jetty-requestlog.xml
CMD java \
  -Dnexus-work=${NEXUS_WORK} -Dnexus-webapp-context-path=${CONTEXT_PATH} -Dapplication-conf=${NEXUS_HOME}/conf \
  -Xms${MIN_HEAP} -Xmx${MAX_HEAP} \
  -cp '/opt/nexus/nexus/conf/:/opt/nexus/nexus/lib/*' \
  ${JAVA_OPTS} \
  org.sonatype.nexus.bootstrap.Launcher ${LAUNCHER_CONF}
