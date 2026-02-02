#IMAGE: Get the base image for Liberty with Java 21
FROM icr.io/appcafe/websphere-liberty:full-java21-openj9-ubi-minimal

# Add MySQL Connector/J 8.4 JDBC driver
RUN mkdir -p /opt/ibm/wlp/usr/shared/resources/mysql
COPY wlp/usr/shared/resources/mysql/mysql-connector-j-8.4.0.jar /opt/ibm/wlp/usr/shared/resources/mysql/
USER root
RUN chown 1001:0 /opt/ibm/wlp/usr/shared/resources/mysql/*.jar
USER 1001

# CONFIG: Add in server.xml
COPY wlp/config/server.xml /config
USER root
RUN chown 1001:0 /config/server.xml
USER 1001

RUN configure.sh

# Add the WAR file
ADD pbw-web/target/plants-by-websphere-jakarta-mysql.war /opt/ibm/wlp/usr/servers/defaultServer/apps
