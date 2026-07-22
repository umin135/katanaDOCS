# KatanaEngine 에셋 & g1m 리서치 — 마스터 인덱스

> **이 문서 세트는 독립(standalone) 지식 베이스다.**
> `_docs/_2_research_docs/` 안의 파일들은 **이 디렉터리 외부의 어떤 파일도 참조하지 않는다.**
> 모든 포맷·알고리즘은 이 문서 안에서 프로즈/표/의사코드로 완결되게 기술한다.
> (외부 도구·구현 파일의 경로를 가리키지 않는다. 지식 자체를 담는다.)

---

## 0. 이 문서가 무엇인가

이 문서 세트는 **코에이 테크모의 KatanaEngine으로 만들어진 게임(특히 Dead or Alive 6: Last Round)의 에셋 포맷**과, 그 위에 커스텀 모드(코드명 **ktmod**)를 얹으려는 시도에서 **리버스 엔지니어링 + 인게임 실측으로 밝혀낸 사실들**을 정리한 것이다.

핵심 대상은 **`g1m` 3D 모델 포맷**, 그중에서도 **NUNO 클로스(천) 시뮬레이션 시스템**이다. 이 부분에 가장 많은 노력과 실측 검증이 들어갔고, 문서의 무게중심도 여기에 있다.

이 문서는 **누군가 이어받아 기여할 수 있도록** 남긴다. 우리(원저자들)는 커스텀 클로스 저작의 마지막 단계(인게임에서 시뮬 정점이 올바른 위치에 오게 하는 것)를 완전히 안정화하지 못한 채 이 문서를 남긴다. 무엇을 알아냈고, 무엇이 아직 미해결인지 [10_open_problems_and_contributing.md](10_open_problems_and_contributing.md)에 정리했다.

---

## 1. 중요 면책 조항 (반드시 읽을 것)

### 1.1 ktmod 기획은 "초기 기획"이며 불안정하다
이 문서에 서술된 **ktmod(모드 패키지/에디터/매니저)의 설계·구동방식은 전부 초기 기획안에 불과하며 불안정하다.** 실제로 구현·검증된 것이 아니라, "이렇게 하면 될 것 같다"는 방향성이다. 특히:

- **ktmod의 `Content/` 메커니즘(에셋을 심볼릭하게 담아 매니저가 설치 시 해석하는 방식)은 무기한 연기(deferred indefinitely)로 결정되었다.** 관련 서술([09_ktmod_design_plan.md](09_ktmod_design_plan.md))은 참고용 방향성일 뿐, 확정된 사양이 아니다.
- 기획 관련 서술은 언제든 바뀔 수 있고, 여기 적힌 대로 구현하면 안 된다. **바뀔 수 있는 기획**과 **입증된 포맷 사실**을 구분해서 읽어라.

### 1.2 포맷 지식은 실측 기반이나, 게임 하나만 검증됨
포맷에 관한 서술은 **바이너리 분석 + 인게임 실측**으로 뒷받침된다. 그러나:

- **완전 검증된 게임은 DOA6: Last Round 하나다.** Wo Long / Nioh / Atelier 계열 등 같은 엔진 계열은 포맷이 유사하나 미검증이다.
- 게임 엔진 내부(물리 시뮬 솔버, globalToFinal 정확한 산출)는 **소스가 없어 관측·추론으로만** 파악했다. 관측 가능한 입출력은 실측했지만, 내부 동작에 대한 일부 설명은 가설이다.

---

## 2. 검증 등급 표기법

문서 전반에서 아래 태그로 신뢰도를 표시한다.

| 태그 | 의미 |
|---|---|
| **[실측]** | 인게임 또는 바이너리에서 직접 관측·재현하여 확인한 사실 |
| **[유도]** | 실측된 여러 사실에서 논리적으로 유도했고, 일관성이 확인된 것 |
| **[가설]** | 아직 인게임/실측으로 확정되지 않은 추정. 반증 가능 |
| **[미해결]** | 우리가 끝내 풀지 못한 문제 |

---

## 3. 읽는 순서 / 문서 구성

에셋 시스템의 큰 그림 → g1m 포맷 → 클로스(핵심) → 기획 → 미해결 순으로 배치했다.

1. [01_katana_engine_asset_system.md](01_katana_engine_asset_system.md) — RDB/RDX/FDATA 인덱스·데이터 계층, IDRK 블록, zlibext, KTID 해시, 오버라이드 전략.
2. [02_asset_reference_chain.md](02_asset_reference_chain.md) — 한 캐릭터 복장이 여러 에셋(모델·텍스처·재질)으로 어떻게 연결되는가. kidsobjdb, 싱글톤 DB, 네임해시 미해결 문제.
3. [03_g1m_container_and_chunks.md](03_g1m_container_and_chunks.md) — g1m 컨테이너 구조와 청크 목록.
4. [04_g1m_skeleton_and_matrices.md](04_g1m_skeleton_and_matrices.md) — G1MS 스켈레톤, jil(로컬↔글로벌), globalToFinal, G1MM 역바인드행렬, oid 본이름.
5. [05_g1m_geometry.md](05_g1m_geometry.md) — G1MG 지오메트리 섹션, 정점버퍼 시맨틱, 조인트 팔레트, 스키닝, 메시그룹.
6. [06_g1m_cloth_system.md](06_g1m_cloth_system.md) — **핵심.** NUNO 클로스 포맷 심층: NUNO1/NUNO3 쌍, 제어점, influences, 앵커, head 파라미터, **forward model**, **클로스 팔레트(physIdx)**, 스파이크 원인.
7. [07_g1m_cloth_authoring.md](07_g1m_cloth_authoring.md) — 입증된 커스텀 클로스 저작 레시피(단계별).
8. [08_g1m_textures_materials_physics.md](08_g1m_textures_materials_physics.md) — g1t 텍스처, kts 재질, sid 메시 레지스트리, SOFT 소프트바디.
9. [09_ktmod_design_plan.md](09_ktmod_design_plan.md) — ktmod 기획(초기·불안정, Content 무기한 연기).
10. [10_open_problems_and_contributing.md](10_open_problems_and_contributing.md) — 미해결 문제, 가설, 기여 안내.

---

## 4. 용어집 (Glossary)

포맷 이해에 필요한 최소 용어. 상세는 각 문서에서 다룬다.

| 용어 | 뜻 |
|---|---|
| **RDB / RDX** | 에셋 인덱스. RDB=파일 위치 메타(file_ktid→위치), RDX=파일명 해시 매핑. |
| **FDATA** | 실제 에셋 데이터 컨테이너. IDRK 블록들의 모음, zlibext 압축. |
| **IDRK** | FDATA 내부의 에셋 단위 블록(`IDRK` 매직). |
| **KTID** | 32비트 에셋 해시. **FileKtid**=에셋 고유(파일명 기반), **TypeKtid**=타입 클래스. |
| **g1m** | KatanaEngine 3D 모델 컨테이너(스켈레톤+지오메트리+물리). |
| **청크(chunk)** | g1m 내부 서브블록(G1MF/G1MS/G1MM/G1MG/NUNO/SOFT 등). |
| **G1MS** | g1m 스켈레톤 청크(본 계층·jil). |
| **jil** | Joint Index List. 로컬본↔글로벌본 ID 매핑 테이블. |
| **로컬 ID** | 이 g1m 내부 본 배열 인덱스(0..N). |
| **글로벌 ID** | 전역(병합) 스켈레톤에서의 본 ID. jil로 로컬↔글로벌 변환. |
| **globalToFinal** | 게임이 글로벌 ID를 최종 조인트 배열 위치로 매핑하는 표(런타임). |
| **G1MM** | 역바인드행렬(inverse bind matrix) 배열 청크. **G1MMIndex**로 색인. |
| **G1MG** | 지오메트리 청크. 정점버퍼·레이아웃·인덱스·서브메시·조인트팔레트·메시그룹 섹션 포함. |
| **조인트 팔레트(bone map)** | 서브메시가 참조하는 (G1MMIndex, physicsIndex, jointIndex) 엔트리 목록. |
| **physicsIndex (cloth)** | 조인트 팔레트 엔트리 필드. **≠0이면 그 팔레트=클로스 물리 팔레트**(게임이 물리 메시로 인식). |
| **NUNO** | 클로스(천) 시뮬 청크. NUNO1/NUNO3(제어점 격자), NUNO4(드라이버 테이블), NUNV(정점 클로스). |
| **제어점(CP, control point)** | 클로스 시뮬 격자의 노드(RichVec4 xyz+w). |
| **influences (NunInfluence)** | 제어점의 위상/구속(P1~P4=이웃 CP 인덱스, P5/P6=rest-length). |
| **앵커(anchor, unkSection)** | 제어점을 본에 고정하는 48바이트 블록(방향벡터+본참조). |
| **parentID** | NUNO 엔트리의 시뮬 구동 본. **글로벌 ID로 저장**됨(→ jil/globalToFinal로 로컬 해석). |
| **리벳(rivet)** | 클로스 디스플레이 메시에서 본에 직접 스킨되는 고정 정점(cpW2=0 플래그). |
| **시뮬 정점(sim vertex)** | 제어점(드라이버 조인트)에 구동되는 자유 정점(cpW2≠0). |
| **oid** | 본 이름 테이블(글로벌 ID→이름 해시→문자열). |
| **sid** | Character.sid. 메시 해시별 렌더/재질/물리 구동 레지스트리(캐릭터 단위 싱글톤). |
| **kts / g1t / kidsobjdb** | 재질(KTS), 텍스처(g1t), 오브젝트 참조 DB. [02](02_asset_reference_chain.md)·[08](08_g1m_textures_materials_physics.md) 참조. |
| **ktmod** | 이 프로젝트의 커스텀 모드 패키지/에디터/매니저 (초기 기획, 불안정). |

---

## 5. 한 줄 요약 (가장 중요한 발견들)

바쁜 독자를 위해, 우리가 입증한 것 중 가장 값진 것들:

- **[실측]** g1m 클로스는 **CP를 `world[드라이버로컬본]` 프레임에 인코딩**하고, 게임은 **`world[globalToFinal[parentID]]`로 배치**한다. 그래서 **parentID는 반드시 글로벌 ID(=l2g[로컬본])**여야 한다. (KOK 샘플 실측: 오차 0.6단위)
- **[실측]** 모든 실제 클로스는 **NUNO1 + NUNO3 쌍**(제어점·influences 완전 동일, skip 구조만 rows/cols에서 파생)으로 저장된다.
- **[실측]** 클로스 디스플레이 메시는 반드시 **physicsIndex=드라이버본인 팔레트**에 바인딩돼야 한다. 아니면(physIdx=0) 게임이 시뮬 정점의 인덱스를 팔레트 슬롯으로 오독해 **정점이 폭발(spike)**한다.
- **[실측]** 클로스 시뮬의 눈에 보이는 흔들림(jiggle)은 **구동 본이 실제로 움직일 때만** 나온다. 고정된 본(가만히 선 다리)에 붙인 클로스는 흔들리지 않는다.

이것들이 왜 그런지, 어떻게 검증했는지는 [06](06_g1m_cloth_system.md)·[07](07_g1m_cloth_authoring.md)에서 상세히 다룬다.
