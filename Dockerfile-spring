# Build stage
FROM gradle:8.10.2-jdk17 AS builder
WORKDIR /app
COPY . .
RUN gradle build --no-daemon

# Production stage
FROM eclipse-temurin:17-jdk-jammy
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar ./
ARG SPRING_PROFILES_ACTIVE
ENV SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}
EXPOSE 8080

# Create a script to find and run the JAR file
RUN echo '#!/bin/sh\n\
JAR_FILE=$(ls -1 *.jar | head -n 1)\n\
if [ -z "$JAR_FILE" ]; then\n\
    echo "No JAR file found in the current directory"\n\
    exit 1\n\
fi\n\
exec java -Dspring.profiles.active=$SPRING_PROFILES_ACTIVE -jar $JAR_FILE' > /app/run.sh && chmod +x /app/run.sh

ENTRYPOINT ["/app/run.sh"]