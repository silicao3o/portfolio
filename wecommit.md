# wecommit-projects

Noshowping(숙박 중고 거래 서비스)

## 사용된 기술
- Language : Python, TypeScript
- Framwork : FastAPI, React-Native
- DB : PostgreSQL
- ETC : Fly.io, GitHub Action, 


## 주요 개발 내용

### 1. 크롤링 자동화 파이프라인 구축 및 성능 최적화
- BeautifulSoup → Selenium → Requests + 병렬처리 과정
- 성능 74% 향상 (103초 → 27초)
- 자동화된 데이터 수집 파이프라인 구축

### 2. 데이터베이스 설계
- 제3정규형 기반 ERD 설계
- User-Merchant 도메인 분리를 통한 보안/성능 최적화
- 확장성을 고려한 테이블 구조 설계

### 3. FastAPI 백엔드 개발
- 처음 접해보는 프레임워크였지만, 기존에 알고 있던 자바/스프링의 구조와 FastAPI의 구조를 비교해서 비슷한 부분끼리 연결지어서 보다 빠르게 지식 습득
- 습득한 지식을 통해 팀원들에게도 똑같이 자바/스프링 구조에 빗대어 설명하여 관련 아티클을 작성하고 개발팀 회의에서 공유
- 엔티티 마이그레이션 작업, 판매글/서비스상품/예약상세 테이블들의 연관관계 적립 및 CRUD기능 개발
- 카카오 알림톡 발송 기능 개발
- 판매자 물건 정기적 체크 배치 로직 구현(판매글 올린 후, 24시간 마다 타 사이트에서 판매되었는지 체크하기 위함)    
- TossPayments 승인 api 구현
- Uploadcare 이미지 보안 기능 구현(URL Signed + JWT토큰 검증)

### 4. React-Native 모바일 개발
- 퍼블리싱 및 기능 개발

### 5. QA
- QA 리스트업 및 테스트 진행

## 트러블 슈팅

### 1. 크롤링 개발 부분

#### 1.1. 크롤링 로직의 무한 루프

**문제 상황**
- 크롤링 로직이 모든 아이템을 찾았음에도 불구하고 계속 실행되는 현상 발견
- 로깅 파일 분석 결과, 필요한 아이템을 모두 수집했음에도 설정된 최대 페이지(125페이지)까지 불필요하게 크롤링 진행

**원인 분석**
- 종료 조건의 부적절한 설정
  - 중고나라의 최대 페이지(125)를 하드코딩하여 종료 조건으로 설정
  - 실제 필요한 아이템 수와 무관하게 최대 페이지까지 크롤링 수행
- 잠재적 문제점
  - 불필요한 서버 요청으로 인한 리소스 낭비
  - 대상 서버에 불필요한 부하 발생 가능성

**해결 방안**
- 크롤링 종료 조건을 다음과 같이 동적으로 변경
  1. 필요한 모든 아이템을 찾았을 때
  2. 더 이상 아이템이 없는 페이지에 도달했을 때

**개선 결과**
- 필요한 아이템을 모두 수집하면 자동으로 크롤링 종료
- 불필요한 페이지 요청 제거로 서버 부하 최소화
- 크롤링 수행 시간 최적화 달성(4시간 -> 1~2시간으로 단축)

#### 1.2. 중고나라 크롤링 성능 개선 과정

**초기 접근 및 문제 발견**
- BeautifulSoup 사용 시도
  - JavaScript 동적 데이터 수집 불가 문제 발생
  - 정적 데이터만 수집 가능한 한계 확인
- Selenium + WebDriver 적용
  - 동적 데이터 수집 문제 해결
  - 성능 이슈 발생 (20개 아이템 기준 103초 소요)
  - 리소스 오버헤드 및 네트워크 오버헤드 발생

**성능 개선 과정**
1. Selenium 제거 및 Requests 도입
   - HTTP 요청 방식으로 변경
   - JavaScript 동적 데이터 수집 문제 재발생
2. 스크립트 파싱 방식 재설계
   - JavaScript 코드 내 데이터 직접 파싱 로직 구현
   - 성능 개선 확인 (20개 기준 103초 -> 58초로 단축)
3. 병렬 처리 구현
   - 동시 다중 요청 처리 로직 추가
   - 최종 성능 대폭 개선 (20개 기준 58초 -> 27초로 단축)

**최종 성능 개선 결과**
- 초기 대비 약 74% 성능 향상 (103초 → 27초)
- 안정적인 데이터 수집과 처리 속도 확보
- 시스템 리소스 사용 효율성 개선

#### 1.3. auto_increment 동작 최적화: INSERT IGNORE 이슈 해결

**문제 상황**
- 크롤링 테이블에 **Keyword** 필드 누락되어 추가하는 수정 작업 발생
- 데이터 크롤링에 관한 log파일로 키워드 및 각 오픈마켓 크롤링된 데이터 수 계산
- SQL 쿼리에서 count값과 max_id의 값이 동일하지 않아서 문제 파악
- 데이터 중복 처리 시 발생한 이슈

```sql
-- 기존 쿼리
INSERT IGNORE INTO items (original_link, title, price) 
VALUES ('example.com/item1', '상품1', 10000);
```

- INSERT IGNORE 사용 시 auto_increment 값 불필요한 증가
- 실제 데이터 수와 ID 최대값의 큰 차이 발생 (약 2만)
- 크롤링 데이터 추적 및 관리의 어려움

**원인 분석**
```sql
-- 문제가 되는 상황 예시
ID  original_link     title   price
1   example.com/1     상품1   10000
2   example.com/2     상품2   20000
3   example.com/3     상품3   30000
5   example.com/4     상품4   40000  -- ID 4가 건너뛰어짐
```

- INSERT IGNORE 실행 시 중복 체크 전에 auto_increment 값 선점
- 중복으로 인한 INSERT 실패 시에도 auto_increment 값 증가

**해결 방안**
```sql
-- auto_increment 동작 최적화 설정
innodb_autoinc_lock_mode = 0
```

**개선 결과**
- auto_increment 값의 연속성 보장
- keyword 필드 추가 후, 누락된 keyword 필드 수정 작업에서 log 파일에 있는 크롤링된 데이터 숫자의 범위를 지정하여 keyword 수정하기 편리
- 데이터 ID와 실제 데이터 수의 일치
- 크롤링된 데이터 추적 및 관리 용이성 향상

### 2. FastAPI 백엔드 개발 부분

#### 2.1. FastAPI 순환 참조 발생
- [FastAPI 순환 참조 해결하기](https://velog.io/@silica_o3o/FastAPI-%EC%88%9C%ED%99%98-%EC%B0%B8%EC%A1%B0-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0)

#### 2.2. 알림톡 서비스 리팩토링

**문제 상황**
- 알림톡 관련 로직이 여러 곳에 분산되어 있음
- 중복 코드 존재
- 순환 참조 문제 발생
- 책임과 의존성이 명확하지 않음

**개선 사항**

1. 모델 구조화
```python
# core/service/solapi/models.py
class ButtonType(Enum):
    WL = "WL"  # 웹링크
    AL = "AL"  # 앱링크

class AlimtalkButton(BaseModel):
    button_type: ButtonType
    button_name: str
    link_mo: str
    link_pc: Optional[str] = None

class BaseAlimtalkRequest(BaseModel):
    template_type: TemplateType
    to: str
    variables: Dict[str, str]
    buttons: Optional[List[AlimtalkButton]] = None
```

2. 서비스 계층 분리
```python
class AlimtalkService:
    def __init__(self, solapi_service: SolapiService):
        self.solapi_service = solapi_service
        
    async def send_product_check(self, request: BaseAlimtalkRequest):
        return await self.solapi_service.send_alimtalk(request)
```

**개선 결과**
1. 코드 구조화
   - 모델, 서비스 계층 분리(프로젝트 패턴인 DDD패턴에 맞게 설계)
   - 책임과 역할 명확

2. 재사용성 및 확장성
   - 공통 모델을 통한 코드 재사용 가능
   - 새로운 알림톡 기능 추가 용이

3. 유지보수성
   - 순환 참조 제거로 코드 안정성 향상
   - 모듈화를 통한 유지보수 용이

#### 2.3. 주문 중복 생성 문제

**문제 상황**
```
173 114 4 ORDER_PENDING          2024-12-11 05:25:56.856 2024-12-11 05:25:56.856 order_1733894751121 제주 한라호텔 양도 300000
172 114 4 ORDER_COMPLETED    25  2024-12-11 05:25:56.497 2024-12-11 05:26:10.581 order_1733894751121 제주 한라호텔 양도 300000
```
- 동일한 origin_order_id에 대해 여러 주문 레코드가 생성됨

**원인**
- order 객체를 save하면서 새롭게 order가 생기면서 order_completed되는 로직으로 인한 중복 생성

**해결**
```python
async def update_order(self, *, order: Order) -> None:
    """Update order"""
    try:
        session.add(order)
        await session.flush()
    except Exception as e:
        raise Exception(f"Failed to update order: {str(e)}")
```
- 기존 엔티티 업데이트 되도록 수정

#### 2.4. TossPayments 중복 생성

**문제 상황**
```
[DEBUG] Response status: 200
[DEBUG] Response status: 400
[DEBUG] Response body: {"code":"ALREADY_PROCESSED_PAYMENT","message":"이미 처리된 결제 입니다."}
[DEBUG] Error: 400: 토스 결제 승인 실패: 이미 처리된 결제 입니다.
[DEBUG] Error: 500: 결제 승인 처리 중 오류 발생: 400: 토스 결제 승인 실패: 이미 처리된 결제 입니다.
```
- 중복으로 결제가 처리됨

**원인**
- Adapter 계층과 Service 계층에 중복되게 tosspayments 승인 api 호출

**해결**
```python
async def update_order(
    request: CreateOrderRequest,
    usecase: OrderUseCase = Depends(Provide[Container.order_service]),
    toss_payment_service: TossPaymentService = Depends(Provide[Container.toss_payment_service]),
    alimtalk_sender: AlimtalkSenderPort = Depends(Provide[Container.alimtalk_service]),
):
    try:
        if (request.payment_type == PaymentType.NORMAL and
            request.payment_status == PaymentStatus.PAYMENT_COMPLETED and
            request.payment_key):
            # 토스 결제 승인 API 호출
            await toss_payment_service.confirm_payment(
                payment_key=request.payment_key,
                order_id=request.origin_order_id,
                amount=request.amount
            )
        command = CreateOrderCommand(**request.model_dump())
        updated_order = await usecase.update_order(command=command)
```
- 계층에 맞게 결제 승인 api 호출을 adapter에만 구현

#### 토스페이먼츠 가상계좌 결제 상태 처리 트러블슈팅
[토스페이먼츠 가상계좌 결제 상태 처리](https://velog.io/@silica_o3o/%ED%86%A0%EC%8A%A4-%ED%8E%98%EC%9D%B4%EB%A8%BC%EC%B8%A0-%ED%8A%B8%EB%9F%AC%EB%B8%94-%EC%8A%88%ED%8C%85)

#### 결제 동시성 문제 처리
[결제 동시성 문제 처리](https://velog.io/@silica_o3o/%EB%8F%99%EC%8B%9C-%EA%B2%B0%EC%A0%9C-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0)

#### 판매글 작성 API 응답시간 줄이기
**문제상황**
- 판매글 작성 시, 로딩이 너무 긴 현상 발견
- Postman API 테스트 시, 응답시간 10.75초

**원인 및 접근방식**
원인에 대해 2가지의 가설을 세우고 접근

1. 이미지 업로드 시간이 크게 작용할 것이다.
2. 작성 시, SQL로그에서 Article 업데이트가 중복으로 일어나기 때문이다.

**해결**

1번 원인 해결방법 
- Article,Reservation,ServiceProduct 생성 트랜잭션에서 이미지 업로드 로직 분리
- 이미지 업로드만 처리하는 API를 따로 개발
- 프론트에서 Article 폼을 받을 때 이미지 업로드만 먼저 호출 후 서버에 이미지 저장
- Reservation 테이블을 생성 후, 업로드 된 이미지 테이블 id를 받아 업데이트

2번 원인 해결방법
- 기존 코드에서 중복으로 처리되던 로직 수정
```python
# 3. ServiceProduct 생성 및 저장
if command.service_product_data:
    service_product = ServiceProduct.create(
        article_id=saved_article.id,
        **command.service_product_data
    )
    service_product = await self.service_product_repository.save(service_product)
    if service_product:
        saved_article.service_product = service_product
        saved_article = await self.repository.save(article=saved_article)

# 4. Reservation 및 ScreenShot 생성 및 저장
if command.reservation_data:
    reservation = Reservation.create(
        article_id=saved_article.id,
        **command.reservation_data
    )
    # 먼저 reservation을 저장하여 ID를 얻습니다
    reservation = await self.reservation_repository.save(reservation)
    if reservation:
        # 5. ScreenShot 생성 및 저장
        if command.reservation_data.get('reservation_image_urls'):
            screen_shot = ScreenShot.create(
                reservation_id=reservation.id,
                image_urls=command.reservation_data['reservation_image_urls']
            )
            await self.screen_shot_repository.save(screen_shot)

        saved_article.reservation = reservation
        saved_article = await self.repository.save(article=saved_article)
```
-> service_product, reservation 생성 후 각각 업데이트 하는 로직 통합

```python
# 3. ServiceProduct 생성 및 저장
if command.service_product_data:
    service_product = ServiceProduct.create(
        article_id=saved_article.id,
        **command.service_product_data
    )
    service_product = await self.service_product_repository.save(service_product)
    if service_product:
        saved_article.service_product = service_product

# 4. Reservation 및 ScreenShot 생성 및 저장
if command.reservation_data:
    reservation = Reservation.create(
        article_id=saved_article.id,
        **command.reservation_data
    )
    # 먼저 reservation을 저장하여 ID를 얻습니다
    logging.warning("##################################### Reservation 저장")
    reservation = await self.reservation_repository.save(reservation)

    if command.reservation_data.get('screenshot_id'):
        # 기존 screenshot을 조회
        screenshot = await self.screen_shot_repository.get_by_id(command.reservation_data['screenshot_id'])

        if screenshot:
            # screenshot의 reservation_id 업데이트
            screenshot = await self.screen_shot_repository.update_reservation_id(
                screenshot_id=screenshot.id,
                reservation_id=reservation.id
            )

    if reservation:
        saved_article.reservation = reservation

# 생성된 Service_Product 및 Reservation 을 Article 에 일괄 반영
saved_article = await self.repository.save(article=saved_article)
```

**성과**
- 2개 적용 시, 약 6.3초까지 API 응답시간 단축(4.4초 단축)

기존 API 응답시간
![image](https://github.com/user-attachments/assets/acfe1d29-75ec-462e-a2bd-3140888afb4b)

로직 수정 후 API 응답시간
![image](https://github.com/user-attachments/assets/27c830c9-738f-4c0a-b280-d27752cce9e6)

**예상 외의 문제**
- discord 웹훅 알림으로 인한 응답시간 증가
- DB서버에서 트래픽이 발생하지 않으면 DB엔진이 다시 불러오는 시간이 필요

**예상 외의 문제 해결**
- Discord 웹훅 문제(0.4초 단축)
웹훅에서 merchant_name을 기존에는 Article에서 Merchant 테이블로 조인 후 가져왔지만 Article 생성할 때 판매자 유효성 검사를 진행하여
조인하지 않고 파라미터로 넘기는 것으로 로직 수정

- DB엔진 실행 문제
트래픽이 발생하면 나오지 않을 문제여서, 헬스체크를 하는 로직으로 해결해야하나 고민하였지만 훗날 자연스럽게 트래픽이 증가하면 해결되는 문제기 때문에 이번에는 헬스 체크 모니터링을 해주는 클라우드로 해결

**최종 결과**

평균 값 약 2.1~2.3초대 API 응답시간 반환
![image](https://github.com/user-attachments/assets/7c481604-3174-411d-9c77-e198e29e2ecc)

