# build the jar
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /build

COPY pom.xml .
RUN mvn dependency:go-offline -q

COPY src ./src
RUN mvn package -DskipTests -q

# run with an JRE image
FROM eclipse-temurin:17-jre

WORKDIR /app

# copy jar
COPY --from=builder /build/target/product-api-*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
