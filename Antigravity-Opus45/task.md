# Substrate Toon BSDF 구현

## 목표
Substrate 렌더링 시스템에 TOON BSDF 타입을 추가하여 Toon 쉐이딩 모델을 올바르게 지원

## 태스크 목록

### 1. BSDF 타입 정의
- [ ] `SubstrateDefinitions.h`에 `SUBSTRATE_BSDF_TYPE_TOON` 추가
- [ ] 필요시 `HEADER_MATERIALMODE_TOON` 추가

### 2. Substrate.ush 수정
- [ ] TOON BSDF 데이터 접근 매크로 정의 (TOON_SHADOWTINT, TOON_SHADOWWEIGHT 등)
- [ ] `GetSubstrateToonBSDF()` 함수 구현
- [ ] `SubstrateGetLegacyShadingModels()` 함수에 TOON case 추가
- [ ] 각종 헬퍼 함수들에 TOON case 추가

### 3. SubstrateLegacyConversion.ush 수정
- [ ] `SubstrateConvertLegacyMaterialStatic()` 함수에 `SHADINGMODELID_TOON` 분기 추가

### 4. SubstrateEvaluation.ush 수정 (핵심)
- [ ] `SubstrateEvaluateBSDFCommon()` 함수에 `SUBSTRATE_BSDF_TYPE_TOON` case 추가
- [ ] Toon 라이팅 로직 구현 (기존 ToonBxDF 참고)

### 5. SubstrateExport.ush 수정
- [ ] GBuffer Export 로직에 TOON case 추가

### 6. 검증
- [ ] 에디터에서 Toon 머티리얼 테스트
