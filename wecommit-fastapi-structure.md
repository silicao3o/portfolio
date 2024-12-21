#FastAPI 프로젝트 아키텍처 가이드

##목차
1. [프로젝트 구조 개요](#프로젝트-구조-개요)
2. [Domain Layer](#1-domain-layer)
   - [UseCase](#usecase)
   - [Entity](#entity)
   - [Command](#command)
   - [Repository Interface](#repository-interface)
3. [Application Layer](#2-application-layer)
   - [Service](#service)
4. [Adapter Layer](#3-adapter-layer)
   - [Input Adapters](#input-adapters)
   - [Output Adapters](#output-adapters)
5. [의존성 주입 설정](#의존성-주입-설정)
6. [데이터 흐름](#데이터-흐름)
7. [주요 특징](#주요-특징)

##프로젝트 구조 개요

```
app/
├── adapter/
│   ├── input/
│   │   └── api/
│   │       └── v1/
│   │           ├── request/
│   │           └── response/
│   └── output/
│       └── persistence/
├── application/
│   ├── service/
│   └── dto/
└── domain/
    ├── command/
    ├── entity/
    ├── repository/
    └── usecase/
```

##1. Domain Layer

도메인 계층은 비즈니스의 핵심 로직과 규칙을 포함합니다.

###UseCase

```python
class ArticleUseCase(ABC):
    def __init__(
        self,
        article_repository,
        service_product_repository,
        reservation_repository,
        merchant_repository,
    ):
        self.article_repository = article_repository
        self.service_product_repository = service_product_repository
        self.reservation_repository = reservation_repository
        self.merchant_repository = merchant_repository

    @abstractmethod
    async def create_article(self, command: CreateArticleCommand) -> "ArticleResponse":
        """게시글 생성"""
        pass

    @abstractmethod
    async def get_articles(
        self,
        *,
        limit: int = 10,
        page: int = 1,
        sort_by: str | None = None,
        keyword: str | None = None,
    ) -> "ArticlesResponse":
        """게시글 목록 조회"""
        pass
```

UseCase는 다음과 같은 역할을 합니다:
- 비즈니스 요구사항을 추상화된 인터페이스로 정의
- 필요한 의존성(repositories)을 명시적으로 선언
- 각 비즈니스 기능의 입력(Command)과 출력(Response) 명세
- 시스템이 수행해야 할 작업을 선언적으로 정의

예를 들어, 게시글 생성 UseCase의 실제 구현은 다음과 같은 작업을 포함합니다:
1. Command를 통해 게시글 생성에 필요한 데이터 수신
2. 판매자(Merchant) 존재 여부 확인
3. 게시글 엔티티 생성
4. 연관된 서비스 상품 정보 생성
5. 예약 정보 생성
6. 저장소에 데이터 저장

### Entity

```python
class Article(BaseEntity):
    __tablename__ = "article"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    title: Mapped[str] = mapped_column(String(50))
    selling_price: Mapped[int] = mapped_column(Integer)

    @classmethod
    def create(cls, title: str, selling_price: int) -> "Article":
        return cls(
            title=title,
            selling_price=selling_price
        )
```

Entity는:
- 비즈니스 도메인 객체 정의
- 데이터베이스 테이블과 매핑
- 핵심 비즈니스 로직 포함

### Command

```python
class CreateArticleCommand(BaseModel):
    title: str
    description: str
    selling_price: int
    service_product_data: Optional[dict] = None
```

Command는:
- 비즈니스 작업 실행을 위한 데이터 구조 정의
- 유효성 검사 규칙 포함
- 비즈니스 작업의 의도를 명확히 표현

### Repository Interface

```python
class ArticleRepo(ABC):
    @abstractmethod
    async def save(self, article: Article) -> "ArticleResponse":
        """Save article"""

    @abstractmethod
    async def get_article_by_id(self, article_id: int) -> Article:
        """Get article by id"""
```

Repository Interface는:
- 데이터 접근 추상화
- 구현에 독립적인 인터페이스 정의
- 도메인 객체의 영속성 관리 방식 정의

## 2. Application Layer

### Service

```python
class ArticleService(ArticleUseCase):
    def __init__(
        self,
        repository: ArticleRepositoryAdapter,
        service_product_repository: ServiceProductRepo
    ):
        self.repository = repository
        self.service_product_repository = service_product_repository

    @Transactional()
    async def create_article(self, command: CreateArticleCommand):
        article = Article.create(
            title=command.title,
            selling_price=command.selling_price
        )
        return await self.repository.save(article=article)
```

Service는:
- 비즈니스 유스케이스 구현
- 트랜잭션 관리
- 도메인 객체 조작
- 여러 도메인 객체 간의 상호작용 조정

## 3. Adapter Layer

### Input Adapters

#### API Routes
```python
@article_router.post("")
async def create_article(
    request: CreateArticleRequest,
    usecase: ArticleUseCase = Depends(Provide[Container.article_service])
):
    command = CreateArticleCommand(**request.model_dump())
    result = await usecase.create_article(command=command)
    return JSONResponse(
        status_code=status.HTTP_200_OK,
        content={"message": "Success", "data": result}
    )
```

API Routes는:
- HTTP 요청 처리
- 요청 데이터 검증
- 유스케이스로 데이터 전달
- 응답 형식 정의

#### Request/Response Models
```python
class CreateArticleRequest(BaseModel):
    title: constr(max_length=50) = Field(..., description="제목")
    selling_price: int = Field(..., description="판매가")
    description: str = Field(..., description="설명")
```

Request/Response Models는:
- API 요청/응답 데이터 구조 정의
- 입력 데이터 검증 규칙 정의
- API 문서화를 위한 메타데이터 제공

### Output Adapters

#### Repository Implementation
```python
class ArticleSQLAlchemyRepos(ArticleRepo):
    async def save(self, article: Article):
        session.add(article)
        await session.flush()
        return article

    async def get_article_by_id(self, article_id: int):
        query = select(Article).where(Article.id == article_id)
        result = await session.execute(query)
        return result.scalar_one_or_none()
```

Repository Implementation은:
- 리포지토리 인터페이스 구현
- 데이터베이스 작업 처리
- ORM 사용
- 데이터 영속성 관리

## 의존성 주입 설정

```python
class Container(containers.DeclarativeContainer):
    article_service = providers.Factory(
        ArticleService,
        repository=providers.Singleton(ArticleRepositoryAdapter)
    )
```

의존성 주입 설정은:
- 의존성 설정 및 관리
- 컴포넌트 생명주기 관리
- 서비스 인스턴스 제공
- 테스트를 위한 모의 객체 주입 용이성 제공

## 데이터 흐름

1. HTTP 요청 → API Router
2. Request DTO → Command 변환
3. Command → Service Layer
4. Service → Domain Logic 실행
5. Domain → Repository
6. Repository → Database
7. Response → Client

## 주요 특징

1. **계층 분리**
   - 각 계층은 명확한 책임을 가짐
   - 의존성 방향이 안쪽으로 향함
   - 관심사의 분리를 통한 유지보수성 향상

2. **비동기 처리**
   - `async/await` 사용
   - 비동기 데이터베이스 작업
   - 높은 동시성 처리 능력

3. **타입 안정성**
   - Pydantic 모델 사용
   - 타입 힌트 활용
   - 런타임 에러 방지

4. **의존성 주입**
   - 느슨한 결합
   - 테스트 용이성
   - 컴포넌트 교체 용이성

5. **트랜잭션 관리**
   - 선언적 트랜잭션 처리
   - 일관성 보장
   - 롤백 자동화

이러한 아키텍처는 다음과 같은 이점을 제공합니다:
- 코드의 모듈성과 재사용성
- 테스트 용이성
- 유지보수 편의성
- 확장성
- 비즈니스 로직의 명확한 분리
