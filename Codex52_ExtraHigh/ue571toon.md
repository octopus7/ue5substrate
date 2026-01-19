# 코덱스 ue5.7 툰 작업

**권장 접근(슬랩(Slab) 안에서 “툰” 플래그로 분기)**

SubstrateEvaluateBSDFCommon()는 BSDFType 기준 스위치라서, 새 BSDF 타입을 추가하기보다 “Slab은 유지 + Toon 플래그/파라미터만 추가”가 수정 범위가 제일 작습니다. (새 SUBSTRATE_BSDF_TYPE_TOON은 샘플링/Env/Export/Visualize 등 스위치가 너무 많아짐)

**어디를 어떻게 바꿀지(핵심 포인트)**

- **셰이딩 모델 추가(엔진/셰이더 공통)**
    - MSM_Toon 추가: EngineTypes.h (line 704)
    - 핀 활성화(툰 파라미터를 어디로 받을지 결정 후 MP_CustomData0/1 등 포함): Material.cpp (line 7879)
    - 셰이더 define 추가: MATERIAL_SHADINGMODEL_TOON를 환경에 뿌림: HLSLMaterialTranslator.cpp (line 2957)
    - 셰이딩 모델 ID 추가(디버그/분기용): ShadingCommon.ush (line 21)
- **Substrate로 “툰” 정보를 안전하게 전달(특히 Single-encoding에서)**
    - BSDF State에 툰 플래그 비트 추가(현재 ___UNUSED___ 3비트 활용): Substrate.ush (line 487)
    - Single-encoding은 BSDF state 전체를 저장하지 않아서(최대 ISTHIN까지만) 플래그가 유실될 수 있음 → **OptimisedLegacyMode를 확장해서 툰을 전달**
        - 상수 추가: SINGLE_OPTLEGACYMODE_TOON(예: 값 6) SubstrateDefinitions.h (line 134)
        - Export(패킹)에서 툰 감지→해당 OptimisedLegacyMode로 별도 패킹: SubstrateExport.ush (line 176) / SubstrateExport.ush (line 559)
        - Unpack에서 OptimisedLegacyMode==TOON이면 툰 플래그/파라미터 복원: Substrate.ush (line 4502)
    - Legacy→Substrate 변환에서 Toon일 때 Slab 만들고 툰 플래그/파라미터 세팅: SubstrateLegacyConversion.ush (line 153)
- **실제 라이팅 분기(사용자 관찰 지점)**
    - Direct/Area/Punctual 공통 분기: SubstrateEvaluation.ush (line 453)
        - case SUBSTRATE_BSDF_TYPE_SLAB: 내부에서 if (BSDF_GETISTOON(...))로 툰 디퓨즈/스펙 밴딩(quantize) 적용
    - Skylight/Reflection(Env)도 같이 맞추려면: SubstrateEvaluateForEnvLight()의 slab 케이스에도 동일한 룰 적용: SubstrateEvaluation.ush (line 1581)

**구현 순서(작업 플로우)**

1. 툰 파라미터 설계(어떤 룩/몇 개 파라미터, 어떤 머티리얼 핀(CustomData0/1 등)로 받을지)
2. MSM_Toon + MATERIAL_SHADINGMODEL_TOON + SHADINGMODELID_TOON 배선(C++/HLSL define)
3. Substrate 전달 경로 구축: Toon 플래그 비트 + SINGLE_OPTLEGACYMODE_TOON + Export/Unpack/LegacyConversion 연결
4. SubstrateEvaluateBSDFCommon()(+ Env)에서 툰 분기 구현
5. (필요 시) Substrate 비활성/구형 경로 대응: ShadingModels.ush (line 1072), ShadingModelsMaterial.ush (line 13) 쪽도 Toon 추가

**확인 질문(이거에 따라 설계가 달라져요)**

- 툰을 **Direct light만** 밴딩하면 되나요, 아니면 **Skylight/Reflection/Lumen GI까지** 밴딩해야 하나요?
- 툰 파라미터는 어떤 핀을 쓰고 싶나요? (예: MP_CustomData0/1=임계값/부드러움, SubsurfaceColor=ShadowTint 등)

툰을 Direct light만 밴딩
이 경우 그림자 리시브는 어떻게 처리되는거야?

그림자 리시브는 “툰 밴딩”이랑 별개로, 라이트 패스에서 만든 FShadowTerms가 최종 Direct 조명에 곱해지면서 처리돼요. Substrate 쪽에선 대체로 이렇게 들어갑니다:

- 슬랩(일반 표면) Direct는 ShadowTerms.SurfaceShadow가 **diffuse/specular에 곱**해짐
    - SubstrateEvaluation.ush (line 535) (Diffuse)
    - SubstrateEvaluation.ush (line 1285) (Specular)
- 그래서 “Direct light만 밴딩”이면 기본 선택지는 2가지예요:
    1. **밴딩은 NoL(또는 BRDF 결과)에만 적용**하고, 그 다음에 * ShadowTerms.SurfaceShadow를 그대로 곱한다 → 그림자 리시브/펜엄브라는 UE 기본 그대로(부드러운 경계 유지)
    2. **툰 그림자까지 원하면** NoL * ShadowTerms.SurfaceShadow(또는 shadow 자체)를 밴딩/threshold 한다 → 펜엄브라도 계단/하드 경계로 바뀜

즉, 아무 추가 처리 안 하면 “그림자 리시브는 기본 그림자 품질 그대로 + 밴딩된 조명에 곱해지는 형태”로 나옵니다.

툰을 Direct light만 밴딩 으로 결정
SubsurfaceColor=ShadowTint 사용
이렇게 결정하고

여분 파라미터는 어떻게 되는거야?

툰을 Direct light만 밴딩 + SubsurfaceColor = ShadowTint로 쓰면, “여분 파라미터”는 결국 **어떤 머티리얼 입력을 툰 전용으로 재해석해서 Substrate BSDF에 저장할지** 문제예요.

- SubsurfaceColor( float3 ) → ShadowTint로 확정 (SSS 기능은 안 쓰는 걸로 보고, SSS 패스/플래그는 켜지 않게 처리)
- 추가로 툰이 보통 필요로 하는 스칼라(예: 밴드 임계값/부드러움/스텝 수)는 UE가 원래 제공하는 **CustomData0/1(2개 float)** 를 툰용으로 쓰는 게 가장 깔끔함
    - 예시 매핑:
        - CustomData0 = ToonShadowThreshold(또는 Steps)
        - CustomData1 = ToonShadowSmoothness(또는 BandOffset)
    - 이 핀들이 툰에서도 보이게 하려면 Material.cpp에서 MP_SubsurfaceColor, MP_CustomData0/1 활성 조건에 MSM_Toon을 추가하면 됨.

Substrate 쪽에서는 조명 패스가 머티리얼 그래프를 못 보니까, 위 값들은 **베이스패스에서 Substrate 데이터로 “패킹”**되어야 해요. 가장 실용적인 설계는:

- BSDF state의 unused 비트 1개를 bHasToonData로 쓰고,
- 슬랩 BSDF에 **uint 1개(32bit)** 를 추가로 저장해서 ShadowTint(rgb) + 스칼라 2개를 같이 넣는 방식(예: PackFuzz처럼 20bit color + 6bit + 6bit).

추가 스칼라가 2개를 넘으면 그때는 “uint를 1개 더 쓰기” vs “정밀도 희생” 중 선택이 필요해요. 어떤 값(Threshold/Steps/Smoothness)을 꼭 머티리얼별로 조절하고 싶은지 정해주면 거기에 맞춰 매핑을 확정할게요.

CustomData0 = ToonShadowThreshold(또는 Steps)
CustomData1 = ToonShadowSmoothness(또는 BandOffset)
말한대로 확정하고 구현 들어가자