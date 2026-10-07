# ---------- Етап 1: збірка (великий образ з Maven) ----------
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src

# Спочатку лише pom.xml: залежності кешуються і не качаються при кожній зміні коду
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

COPY src ./src
RUN mvn -B -q clean package -DskipTests

# ---------- Етап 2: запуск (малий образ лише з JRE) ----------
FROM eclipse-temurin:17-jre

# Не запускаємо застосунок від root
RUN useradd --system --uid 10001 --no-create-home appuser
WORKDIR /app
COPY --from=build --chown=10001:10001 /src/target/*.war /app/app.war

USER 10001
EXPOSE 8080

# Ліміт памʼяті Java задається у відсотках від ліміту контейнера
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75"
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.war"]
