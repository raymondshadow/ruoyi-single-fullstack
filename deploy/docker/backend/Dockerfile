FROM maven:3.9.11-eclipse-temurin-17 AS builder

WORKDIR /build

COPY ruoyi-server /build

RUN mvn -pl ruoyi-admin -am -DskipTests package

FROM eclipse-temurin:17-jre

WORKDIR /app

ENV TZ=Asia/Shanghai
ENV JAVA_OPTS=""

COPY --from=builder /build/ruoyi-admin/target/ruoyi-admin.jar /app/ruoyi-admin.jar

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/ruoyi-admin.jar"]
