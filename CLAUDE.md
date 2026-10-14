# CLAUDE.md

Spring Boot 기반 Petclinic 읽기 전용 REST API 프로젝트 (JPA + H2).

## 기술 스택

- Java 17 (Gradle toolchain), Gradle 9.7.1 (Kotlin DSL, wrapper 사용)
- Spring Boot 4.1.1: `data-jpa`, `webmvc`, `thymeleaf`
- springdoc-openapi 3.1.1 (Swagger UI)
- H2 인메모리 DB (`runtimeOnly`), 개발 시 `h2console`, `devtools`
- 테스트: JUnit Platform (`spring-boot-starter-*-test`)
- 그룹/패키지: `com.example.petclinic`, 아티팩트 `petclinic-jpa`

## 명령어

```
./gradlew bootRun     # 실행
./gradlew build       # 빌드 + 테스트
./gradlew test        # 테스트만
```

## 디렉터리 구조

```
src/main/java/com/example/petclinic/
├── PetclinicApplication.java   # 진입점
├── LoadDatabase.java           # CommandLineRunner로 시작 시 샘플 데이터 적재
├── controller/                 # @RestController, 기본 경로 /api/v1/...
│   └── response/               # 응답 DTO (*Dto)
├── service/                    # @Service, 클래스 레벨 @Transactional(readOnly = true)
├── repository/                 # JpaRepository 인터페이스
└── entity/                     # JPA 엔티티
src/main/resources/application.yml   # 현재 비어 있음 (기본 설정 사용)
src/test/java/.../PetclinicApplicationTests.java  # contextLoads 테스트만 존재
```

계층 흐름: `Controller → Service → Repository → Entity`. 각 도메인(Member, Owner, Pet, PetType, Speciality, Vet, Visit)마다 Controller/Service/Repository/Dto가 1:1로 대응한다. (Role, VetSpeciality는 Repository와 Entity만 있음)

## 도메인 모델

- `Person` (`people` 테이블): SINGLE_TABLE 상속, 구분 컬럼 `dtype`
  - `Owner` (`OWNER`): address, city, telephone
  - `Vet` (`VET`)
- `Pet` (`pets`): N:1 → `PetType`(`types`), N:1 → `Owner`
- `Visit` (`visit`): N:1 → `Pet`
- `VetSpeciality` (`vet_specialities`): `Vet`와 `Speciality`(`specialities`)의 연결 엔티티 (명시적 조인 엔티티, @ManyToMany 아님)
- `Member` (`members`, PK `username`) 1 : N `Role` (`roles`)
- 모든 연관관계는 `FetchType.LAZY`, 날짜는 `LocalDateTime`

## REST 엔드포인트 (모두 GET만 존재)

| 경로 | 설명 |
|---|---|
| `/api/v1/owners`, `/{id}` | 소유자 |
| `/api/v1/pets`, `/{id}` | 반려동물 |
| `/api/v1/types`, `/{id}` | 반려동물 종류 |
| `/api/v1/vets`, `/{id}` | 수의사 |
| `/api/v1/specialities`, `/{id}` | 전문 분야 |
| `/api/v1/visits`, `/{id}` | 진료 기록 |
| `/api/v1/members`, `/{username}` | 회원 |

## 코드 컨벤션

- 의존성은 생성자 주입 (Lombok 미사용), 엔티티는 수동 getter/setter
- 엔티티 → DTO 변환은 컨트롤러에서 직접 수행 (별도 mapper 없음)
- 서비스의 조회 실패는 `NoSuchElementException("... does not exist")` 로 처리 (`verifyXxxExists` 패턴)
- 엔티티 PK는 `@GeneratedValue(strategy = IDENTITY)` (Member 제외)

## 참고 사항

- `LoadDatabase`가 앱 시작마다 샘플 데이터(수의사 6, 종류 6, 소유자 10, 펫 13, 진료 4, 관리자 계정 `admin`)를 삽입한다. 비밀번호는 `{noop}` 평문 접두어 형식이므로 개발용 전용.
- Spring Security는 의존성에 없다 (Member/Role은 데이터 모델만 존재).
- `.gradle/`, `.idea/`는 로컬 생성물. git 저장소는 아직 초기화되지 않았다.
