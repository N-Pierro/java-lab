# Create a base stage 
FROM eclipse-temurin:21-jdk-jammy as base 
WORKDIR /build
COPY --chmod=0775 mvnw mvnw 
COPY .mvn/ .mvn/

# Create a test stage 
FROM base as test
WORKDIR /build
COPY ./src src/
RUN --mount=type=bind,source=pom.xml,target=pom.xml \
    --mount=type=cache,target=/root/.m2 \
    ./mvnw test 

########################################################################################################
# Create a stage for resolving ans downloading dependecies 
# Above stages are added for test.
# This still works without the above stages, uncommenting the From deps stage 
#FROM eclipse-temurin:21-jdk-jammy as deps 
FROM test as deps

WORKDIR /build
# Commenting out the COPY steps since they are in the base and test stages
# Copy the mvnw wrapper with executable permission 
#COPY --chmod=0755 mvnw mvnw
#COPY .mvn/ .mvn/

# Download dependecies as a seperate step to take advantage of Docker's caching.
# Leverage a cache mount to /root/.m2 so that subsequent builds don't have to 
# re-download packages.

RUN --mount=type=bind,source=pom.xml,target=pom.xml \
    --mount=type=cache,target=/root/.m2 \
    ./mvnw dependency:go-offline -DskipTests

##################################################################################################
# Create a stage for building the application based on the stage with downloaded dependencies
FROM deps as package 

WORKDIR /build

COPY ./src src/
RUN --mount=type=bind,source=pom.xml,target=pom.xml \
    --mount=type=cache,target=/root/.m2 \ 
    ./mvnw package -DskipTests && \
    mv target/$(./mvnw help:evaluate -Dexpression=project.artifactId -q -DforceStdout)-$(./mvnw help:evaluate -Dexpression=project.version -q -DforceStdout).jar target/app.jar

##################################################################################################################

# Create a stage for extracting the application into seperate layers.

FROM package as extract 

WORKDIR /build 

RUN java -Djarmode=layertools -jar target/app.jar extract --destination target/extracted 

###################################################################################################################

FROM extract as development
WORKDIR /build
RUN cp -r /build/target/extracted/dependencies/. ./
RUN cp -r /build/target/extracted/spring-boot-loader/. ./
RUN cp -r /build/target/extracted/snapshot-dependencies/. ./
RUN cp -r /build/target/extracted/application/. ./
ENV JAVA_TOOL_OPTIONS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000
CMD [ "java", "-Dspring.profiles.active=postgres", "org.springframework.boot.loader.launch.JarLauncher" ]

#####################################################################################################################
FROM eclipse-temurin:21-jre-jammy AS final 

# Create a non-privileged user that the app will run under
# See https://docs.docker.com/dockerfile 

ARG UID=10001
RUN adduser \
    --disabled-password \ 
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser
USER appuser 

# Copy the executable from the "package" stage.
COPY --from=extract build/target/extracted/dependencies/ ./
COPY --from=extract build/target/extracted/spring-boot-loader/ ./
COPY --from=extract build/target/extracted/snapshot-dependencies/ ./
COPY --from=extract build/target/extracted/application/ ./

EXPOSE 8080

ENTRYPOINT [ "java", "-Dspring.profiles.active=postgres", "org.springframework.boot.loader.launch.JarLauncher" ]

