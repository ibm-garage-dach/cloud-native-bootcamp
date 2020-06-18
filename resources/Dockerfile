# 1. Package the application using a maven-based builder image

FROM maven:3.6-jdk-8 as builder

COPY . .
RUN mvn package


# 2. Build the Open Liberty image

FROM openliberty/open-liberty:javaee8-ubi-min

ENV INSTALL_DIR /opt/ol/wlp/
ENV SERVER_DIR /opt/ol/wlp/usr/servers/defaultServer/

# Add MySQL-Connector
RUN curl -o ${INSTALL_DIR}lib/mysql-connector-java-8.0.16.jar https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.16/mysql-connector-java-8.0.16.jar
COPY liberty-mysql/mysql-driver.xml ${SERVER_DIR}configDropins/defaults/

# Copy WAR-file from builder image
COPY --from=builder target/health-api.war ${SERVER_DIR}apps

# Copy configs
COPY liberty/server.xml $SERVER_DIR
COPY liberty/jvm.options $SERVER_DIR

# Expose port 9080 where the application runs on
EXPOSE 9080
