FROM amazoncorretto:20-alpine-jdk AS build

WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon # Or 'mvn clean package'

FROM amazoncorretto:20-alpine
WORKDIR /app
# Copy the compiled jar from the build stage
RUN find . -name "*.jar"

COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
