FROM bellsoft/liberica-openjdk-alpine:21

# bash 및 coreutils 설치
RUN apk add --no-cache bash coreutils

# [수정] 현재 경로(./)에 있는 jar 파일을 복사
COPY ./*.jar app.jar

ENTRYPOINT ["java", "-jar", "/app.jar"]
