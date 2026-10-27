# ----------- Phase 1: Build the WAR with Maven (Java 11)
FROM maven:3.8-eclipse-temurin-11 AS build
RUN apt-get update && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN git clone --depth 1 -b main https://github.com/hkhcoder/vprofile-project.git .
RUN mvn package -DskipTests -B -ntp

# ---------- Phase 2: Running it on Tomcat 9 ----------
FROM tomcat:9.0-jdk11-temurin
RUN rm -rf /usr/local/tomcat/webapps/*
COPY --from=build /src/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
