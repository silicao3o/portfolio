# wecommit-projects

Noshowping 프로젝트(숙박 중고 거래 서비스)

1. 크롤링 자동화 파이프라인 구축 및 성능 최적화
   - BeautifulSoup → Selenium → Requests + 병렬처리 과정
   - 성능 74% 향상 (103초 → 27초)
   - 자동화된 데이터 수집 파이프라인 구축

2. 데이터베이스 설계
   - 제3정규형 기반 ERD 설계
   - User-Merchant 도메인 분리를 통한 보안/성능 최적화
   - 확장성을 고려한 테이블 구조 설계

3. FastAPI 백엔드 개발
   - 처음 접해보는 프레임워크였지만, 기존에 알고 있던 자바/스프링의 구조와 FastAPI의 구조를 비교해서 비슷한 부분끼리 연결지어서 보다 빠르게 지식 습득
   - 습득한 지식을 통해 팀원들에게도 똑같이 자바/스프링 구조에 빗대어 설명하여 관련 아티클을 작성하고 개발팀 회의에서 공유
   - 엔티티 마이그레이션 작업, 판매글/서비스상품/예약상세 테이블들의 연관관계 적립 및 CRUD기능 개발
   - 카카오 알림톡 발송 기능 개발
   - 판매자 물건 정기적 체크 배치 로직 구현(판매글 올린 후, 24시간 마다 타 사이트에서 판매되었는지 체크하기 위함)    
   - TossPayments 승인 api 구현
   - Uploadcare 이미지 보안 기능 구현(URL Signed + JWT토큰 검증)
     
4. Reach-Native 모바일 개발
   - Screen 및 API 연동 개발
5. Next-auth
  - 카카오 로그인 구현
6. QA
   - QA 리스트업 및 테스트 진행


# 트러블 슈팅

  ## 크롤링 개발 부분
  - 크롤링 로직의 무한 루프
    - 문제 상황
        - 크롤링 로직이 모든 아이템을 찾았음에도 불구하고 계속 실행되는 현상 발견
        - 로깅 파일 분석 결과, 필요한 아이템을 모두 수집했음에도 설정된 최대 페이지(125페이지)까지 불필요하게 크롤링 진행
    - 원인 분석
        - 종료 조건의 부적절한 설정
            - 중고나라의 최대 페이지(125)를 하드코딩하여 종료 조건으로 설정
            - 실제 필요한 아이템 수와 무관하게 최대 페이지까지 크롤링 수행
        - 잠재적 문제점
            - 불필요한 서버 요청으로 인한 리소스 낭비
            - 대상 서버에 불필요한 부하 발생 가능성
    - 해결 방안
        - 크롤링 종료 조건을 다음과 같이 동적으로 변경
            1. 필요한 모든 아이템을 찾았을 때
            2. 더 이상 아이템이 없는 페이지에 도달했을 때
    - 개선 결과
        - 필요한 아이템을 모두 수집하면 자동으로 크롤링 종료
        - 불필요한 페이지 요청 제거로 서버 부하 최소화
        - 크롤링 수행 시간 최적화 달성(4시간 -> 1~2시간으로 단축)

- 중고나라 크롤링 성능 개선 과정
    - 초기 접근 및 문제 발견
        - BeautifulSoup 사용 시도
            - JavaScript 동적 데이터 수집 불가 문제 발생
            - 정적 데이터만 수집 가능한 한계 확인
        - Selenium + WebDriver 적용
            - 동적 데이터 수집 문제 해결
            - 성능 이슈 발생 (20개 아이템 기준 103초 소요)
                → 리소스 오버헤드 및 네트워크 오버헤드 발생
    - 성능 개선 과정
        - Selenium 제거 및 Requests 도입
            - HTTP 요청 방식으로 변경
            - JavaScript 동적 데이터 수집 문제 재발생
        - 스크립트 파싱 방식 재설계
            - JavaScript 코드 내 데이터 직접 파싱 로직 구현
            - 성능 개선 확인 (20개 기준 103초 -> 58초로 단축)
        - 병렬 처리 구현
            - 동시 다중 요청 처리 로직 추가
            - 최종 성능 대폭 개선 (20개 기준 58초 -> 27초로 단축)
    - 최종 성능 개선 결과
        - 초기 대비 약 74% 성능 향상 (103초 → 27초)
        - 안정적인 데이터 수집과 처리 속도 확보
        - 시스템 리소스 사용 효율성 개선

          
- auto_increment 동작 최적화: INSERT IGNORE 이슈 해결
    - 문제 상황
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
              
    - 원인 분석
        
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
    
    - 해결 방안
        
        ```sql
        - auto_increment 동작 최적화 설정
        innodb_autoinc_lock_mode = 0
        ```
        
    - 개선 결과
        - auto_increment 값의 연속성 보장
          - keyword 필드 추가 후, 누락된 keyword 필드 수정 작업에서 log 파일에 있는 크롤링된 데이터 숫자의 범위를 지정하여 keyword 수정하기 편리
        - 데이터 ID와 실제 데이터 수의 일치
        - 크롤링된 데이터 추적 및 관리 용이성 향상

## FastAPI 백엔드 개발 부분
  - [FastAPI 순환 참조 발생](https://velog.io/@silica_o3o/FastAPI-%EC%88%9C%ED%99%98-%EC%B0%B8%EC%A1%B0-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0)

**알림톡 서비스 리팩토링**
 - 문제 상황
   - 알림톡 관련 로직이 여러 곳에 분산되어 있음
   - 중복 코드 존재
   - 순환 참조 문제 발생
   - 책임과 의존성이 명확하지 않음
 - 해결 과정
   1. 모델 구조화
      - 알림톡에 필요한 값들 기본 모델로 정의
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
    - 알림톡 서비스 로직 통합
    ```python
      class AlimtalkService:
    def __init__(self, solapi_service: SolapiService):
        self.solapi_service = solapi_service

    async def send_product_check(self, request: BaseAlimtalkRequest):
        return await self.solapi_service.send_alimtalk(request)
   ```
- 개선된 점
  1. 코드 구조화
   - 모델, 서비스 계층 분리(프로젝트 패턴인 DDD패턴에 맞게 설계)
   - 책임과 역할 명확
  2. 재사용성 및 확장성
   - 공통 모델을 통한 코드 재사용 가능
   - 새로운 알림톡 기능 추가 용이
  3. 유지보수성
   - 순환 참조 제거로 코드 안정성 향상
   - 모듈화를 통한 유지보수 용이

**주문 중복 생성 문제**

1. 초기 상황
```
173	114	4	ORDER_PENDING			2024-12-11 05:25:56.856	2024-12-11 05:25:56.856	order_1733894751121	제주 한라호텔 양도	300000
172	114	4	ORDER_COMPLETED		25	2024-12-11 05:25:56.497	2024-12-11 05:26:10.581	order_1733894751121	제주 한라호텔 양도	300000
```
- 동일한 origin_order_id에 대해 여러 주문 레코드가 생성됨 

2. 원인
- order 객체를 save하면서 새롭게 order가 생기면서 order_completed되는 로직
 -> 중복 생성 원인

3. 해결
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


**TossPayments 중복 생성**
 1. 문제 상황
```
[DEBUG] Response status: 200
[DEBUG] Response status: 400
[DEBUG] Response body: {"code":"ALREADY_PROCESSED_PAYMENT","message":"이미 처리된 결제 입니다."}
[DEBUG] Error: 400: 토스 결제 승인 실패: 이미 처리된 결제 입니다.
[DEBUG] Error: 500: 결제 승인 처리 중 오류 발생: 400: 토스 결제 승인 실패: 이미 처리된 결제 입니다.
```
- 중복으로 결제가 처리 됨.

2. 원인
   Adapter 계층과 Service 계층에 중복되게 tosspayments 승인 api 호출

3. 해결
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
        ...
   ```
- 계층에 맞게 결제 승인 api 호출 adapter에만 구현


   
    
