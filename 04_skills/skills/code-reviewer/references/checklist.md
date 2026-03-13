# Code Review Checklist

## Python (Django/FastAPI)
- [ ] Type Hinting이 적절히 적용되었는가?
- [ ] List Comprehension이 지나치게 복잡하지 않은가?
- [ ] Django: `select_related`나 `prefetch_related`를 사용하여 N+1 문제를 방지했는가?
- [ ] FastAPI: `Depends`를 이용한 의존성 주입이 깔끔한가?

## Frontend (React/TypeScript)
- [ ] 컴포넌트가 너무 크지 않고 재사용 가능한가?
- [ ] `useEffect`의 의존성 배열이 올바르게 설정되었는가?
- [ ] Props Drilling이 심하지 않은가? (Context API나 상태 관리 라이브러리 고려)

## Common
- [ ] Error Handling: 예외 처리가 구체적이며 에러 메시지가 유용한가?
- [ ] Testing: 유닛 테스트나 통합 테스트가 포함되었는가?
- [ ] Documentation: 복잡한 로직에 대한 주석이 충분한가?
