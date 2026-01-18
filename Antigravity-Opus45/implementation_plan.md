# Substrate TOON BSDF 구현 계획

Substrate 렌더링 시스템에서 `SHADINGMODELID_TOON`을 올바르게 처리하기 위해 새로운 BSDF 타입 `SUBSTRATE_BSDF_TYPE_TOON`을 추가합니다.

## Custom Data 사용
| 채널 | 용도 | 상태 |
|------|------|------|
| RGB | Shadow Tint Color | 구현 |
| A | Shadow Weight | 예약 (미구현) |

---

## Proposed Changes

### Component 1: BSDF 타입 정의

#### [MODIFY] [SubstrateDefinitions.h](file:///e:/sourcetree_repo/ue570toon/Engine/Shaders/Shared/SubstrateDefinitions.h)
- 38라인 부근에 새 BSDF 타입 상수 추가:
```hlsl
#define SUBSTRATE_BSDF_TYPE_TOON  6
```
- 131라인 부근에 MaterialMode 추가:
```hlsl
#define HEADER_MATERIALMODE_TOON  7
```

---

### Component 2: BSDF 생성 및 접근 함수

#### [MODIFY] [Substrate.ush](file:///e:/sourcetree_repo/ue570toon/Engine/Shaders/Private/Substrate/Substrate.ush)

**데이터 접근 매크로 추가** (HAIR 매크로 참고):
```hlsl
// TOON BSDF 데이터 접근 매크로
#define TOON_BASECOLOR(X)      X.State[0].xyz
#define TOON_DIFFUSEALBEDO(X)  X.State[0].xyz
#define TOON_F0(X)             X.State[1].xyz
#define TOON_ROUGHNESS(X)      X.State[1].w
#define TOON_SHADOWTINT(X)     X.State[2].xyz  // CustomData.rgb
#define TOON_SHADOWWEIGHT(X)   X.State[2].w    // CustomData.a (예약)
```

**`GetSubstrateToonBSDF()` 함수 추가**:
- 기존 `GetSubstrateHairBSDF()` 패턴 참고
- 파라미터: BaseColor, Roughness, ShadowTint, ShadowWeight, Emissive, SharedLocalBasisIndex

**헬퍼 함수에 TOON case 추가**:
- `SubstrateGetBSDFDiffuseColor()` (~3623라인)
- `SubstrateGetBSDFWorldNormal()` (~3635라인)
- `SubstrateSetBSDFDiffuseColor()` (~3647라인)
- `SubstrateGetBSDFRoughness()` (~3777라인)
- `SubstrateGetLegacyShadingModels()` (~3902라인)

---

### Component 3: Legacy 변환 로직

#### [MODIFY] [SubstrateLegacyConversion.ush](file:///e:/sourcetree_repo/ue570toon/Engine/Shaders/Private/Substrate/SubstrateLegacyConversion.ush)

**`SubstrateConvertLegacyMaterialStatic()` 함수 수정** (594~624라인 부근):
```hlsl
else if (ShadingModel == SHADINGMODELID_TOON)
{
    Out = GetSubstrateToonBSDF(
        BaseColor,
        Roughness,
        SubSurfaceColor,  // Shadow Tint로 사용
        Opacity,          // Shadow Weight로 사용 (예약)
        Emissive,
        SharedLocalBasisIndex);
}
```

---

### Component 4: BSDF 평가 로직 (핵심)

#### [MODIFY] [SubstrateEvaluation.ush](file:///e:/sourcetree_repo/ue570toon/Engine/Shaders/Private/Substrate/SubstrateEvaluation.ush)

**`SubstrateEvaluateBSDFCommon()` 함수에 TOON case 추가** (1131라인 부근, HAIR case 참고):

```hlsl
case SUBSTRATE_BSDF_TYPE_TOON:
{
    float3 DiffuseColor = TOON_DIFFUSEALBEDO(BSDFContext.BSDF);
    float3 F0           = TOON_F0(BSDFContext.BSDF);
    float  Roughness    = TOON_ROUGHNESS(BSDFContext.BSDF);
    float3 ShadowTint   = TOON_SHADOWTINT(BSDFContext.BSDF);
    
    // Toon Diffuse: 기존 ToonBxDF 로직 적용
    // - Step 기반 NoL 계산
    // - ShadowTint를 그림자 영역에 적용
    
    // Toon Specular: Stylized 하이라이트
    
    Sample.ThroughputV         = OpaqueBSDFThroughput;
    Sample.TransmittanceAlongN = OpaqueBSDFThroughput;
    break;
}
```

---

### Component 5: GBuffer Export

#### [MODIFY] [SubstrateExport.ush](file:///e:/sourcetree_repo/ue570toon/Engine/Shaders/Private/Substrate/SubstrateExport.ush)

**BSDF 타입별 Export 로직에 TOON case 추가** (246, 833라인 부근):
- TOON BSDF에서 GBuffer로 데이터 내보내기

---

## Verification Plan

### Manual Verification
1. 엔진 빌드 완료 후 에디터 실행
2. Toon Shading Model을 사용하는 머티리얼 생성
3. 다음 항목 확인:
   - Toon 스타일 렌더링이 정상 동작하는지
   - Shadow Tint Color가 그림자 영역에 적용되는지
   - 검은색 아티팩트가 발생하지 않는지

> [!IMPORTANT]
> 엔진 빌드는 사용자가 직접 진행합니다 (빌드 규칙에 따름)
