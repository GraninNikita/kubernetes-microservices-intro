# Build stage
FROM gradle:8-jdk21 AS build
WORKDIR /app
COPY . .
RUN gradle :songs-service:build -x test --no-daemon

# Runtime stage
FROM amazoncorretto:21
WORKDIR /app
COPY --from=build /app/songs-service/build/libs/*.jar /app/app.jar

CMD ["java", "-jar", "app.jar"]