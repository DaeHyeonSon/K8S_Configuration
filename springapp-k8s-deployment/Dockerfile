# 베이스 이미지 설정 (Java 11)
FROM openjdk:17-slim

# JAR 파일을 컨테이너 내부에 복사
COPY SpringApp-0.0.1-SNAPSHOT.jar /app/SpringApp.jar

# 워킹 디렉터리 설정
WORKDIR /app

# 애플리케이션 포트 공개
EXPOSE 8080

# 애플리케이션 실행 명령
ENTRYPOINT ["java", "-jar", "SpringApp.jar"]
