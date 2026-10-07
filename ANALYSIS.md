# 学园终端 별도 한국어 패치 설계 분석

작성일: 2026-10-07. 분석 대상: Workshop `3812418766`, `info.ini` 버전 `0.4.2`.

이번 작업은 **정적 분석과 설계 문서 작성까지만** 수행했다. 원본 모드와 DLL, 데이터, 파일명은 변경하지 않았고, 번역문이나 한국어 패치 코드·DLL을 만들지 않았다. 아래 패치 지점과 폴더 구조는 향후 구현 제안이다. 게임 내 작동을 검증한 결과와 구분해야 한다.

## 1. 결론과 분석 근거

별도 모드로 한국어를 표시하는 구조가 가능하다. 다만 원본 전체가 게임 Localization을 거치는 구조는 아니다. 주 UI는 원본의 `学园终端.Ui.Text/Label`과 직접적인 `TMP_Text.text` 대입으로 생성되며, 맵 안내에는 별도의 IMGUI 출력이 있다. 따라서 **외부 번역 사전 + 원본 모드에 한정한 표시 단계의 런타임 패치 + 한국어 폰트 fallback**을 중심으로 하고, 실제 Localization key가 확인된 맵 이름에는 override를 사용한다.

가장 중요한 제약은 `Name`, 중국어 분류명, 파일명도 내부 식별자로 쓰인다는 점이다. 예를 들어 선물 이름으로 가방 저장 키를 만들고, 상점 분류 이름으로 상품을 필터링하며, 구매 상세 팝업은 이미 표시한 제목을 다시 읽어 분기한다. JSON 로딩 시 이름을 일괄 번역하거나 모든 TMP 문자열을 전역 치환하면 원본 로직이 깨질 수 있다.

분석 근거:

- 저장소는 초기화된 Git checkout이지만 원본 모드 파일이 들어 있지 않았다. 기존 `ANALYSIS.md`는 이번 분석 결과로 처음부터 다시 작성했다.
- 이번에 제공된 `3812418766.7z.001`부터 `.022`까지 22개 분할 파일을 결합해 검사·추출했다. 결합 크기는 532,974,071바이트, 압축 검사 결과 오류가 없었다. 이전 ZIP 분할본을 섞어 사용하지 않았다.
- 압축에는 디렉터리 346개, 파일 1,371개가 있다. 원본은 저장소 밖 `/workspace/scratch/mod-investigation/original/3812418766`에 추출했다.
- `学园终端.dll`을 ILSpy로 디컴파일하고 .NET 메타데이터로 주요 타입, 시그니처, 메서드 token을 대조했다. 원본 DLL을 실행하거나 수정하지 않았다.
- JSON 97개를 목록화하고 JSON5로 모두 파싱했다. 일부 파일에 주석이 있어 엄격한 JSON 파서만으로 검사하면 오류가 난다. 실제 로더별 주석 처리도 따로 확인했다.
- CSV, TXT, 문서 및 이미지·폰트·AssetBundle의 구조를 조사했다. 이미지 일부는 직접 보았지만 모든 이미지 OCR이나 동영상 프레임 검사는 수행하지 않았다.
- 원본 파일 전체의 SHA-256 목록을 조사 전에 만들었다. 조사 후 같은 목록과 대조했고 **1,371개 모두 변경이 없었다**.

원본 DLL의 SHA-256:

`cfbdb28e94c7463f8fcffc5324171f92ce5b32c2de517bf512a3270eeb2647c9`

어셈블리 버전은 `1.0.0.0`이다. `info.ini` 버전 및 코드의 오래된 로그 문구와 혼동하지 않는다. 디컴파일 결과는 분석용이며, 참조 DLL 부재로 일부 타입 표현이 부정확한 C# 프로젝트를 그대로 재빌드하는 방식은 권장하지 않는다.

## 2. 원본 모드 구조

주요 구조는 다음과 같다. 경로 표기는 원본 이름을 보존했다.

```text
3812418766/
  info.ini
  学园终端.dll
  characters.json / weapons.json / shop.json / buy_pyroxene.json
  config.json / crafting.json / cafe.json / board.json / upgrade.json
  好感礼物.json / 装备强化.json
  角色/                 캐릭터별 이미지·음성·스킬·효과 데이터
  升级素材/             등급·종류·학교별 스킬 강화 재료
  装备/ / 装备素材/       장비, 설계도, 강화 재료
  好感礼物素材/ / 商城素材/ / 道具素材/
  抽卡x招募/             모집 설정·배너
  任务/                 quests.json 등
  商店/商店分类/          분류별 JSON
  邮件/ / 公告/          메일·공지와 템플릿
  业务区/ / 新场景地图/ / 学园背景/
  assets/ / 主菜单图标/ / 表情包/ / BGM/
  fonts/学园终端字体.ttf
  fonts/学园终端字体.bundle
  技能模板/              효과 템플릿, CSV/JSON 참고 자료
```

확장자별 파일 수: PNG 857, OGG 248, JSON 97, TXT 93, WEBP 22, JPG 12, GIF 11, MD 9, CSV 6, DLL 3, WAV 3, MP4 2, BUNDLE 1, TTF 1, INI 1, PDF 1. 그 밖에 HTTP 백업, 로그, `.new`, 확장자 없는 파일이 각각 1개다. 이미지 확장자 파일은 총 902개다.

DLL의 주요 사용자 코드 namespace는 `学园终端`, `DuckovMapStudio`, `DuckovMapPlayer`다. 세 가지 기능이 한 어셈블리에 함께 들어 있어 메인 화면만 조사하면 맵의 Localization·IMGUI 경로를 놓치게 된다.

일부 경로는 압축 안에서도 `╩╟▀Σ`, `≡ñφ┬═∩█⌐UI`, `▀┬∩┴/▀┬∩┴═φ╥÷.json`, `█╬°╨/╘│╬².json`처럼 보인다. 추출 이름만 보고 중국어 이름을 추정하여 고치지 않았다. 코드가 요구하는 정확한 경로와 다른 파일은 런타임 사용 여부를 따로 검증해야 한다.

## 3. DLL 진입점과 초기화 흐름

### 3.1 진입점

진입점은 **`学园终端.ModBehaviour : Duckov.Modding.ModBehaviour`**다. 주요 lifecycle은 `OnAfterSetup()`과 `OnBeforeDeactivate()`다.

`OnAfterSetup()`의 주요 흐름:

1. 어셈블리 위치로 `ModDir`를 구하고 `ModPaths.Init`, `Log.Init`, 기존 저장 데이터 migration을 수행한다.
2. `Assets.Init`, `Config.Load`, `BaPortraits.Init/Reindex`, `BaVoice.Init/Reindex`를 실행한다.
3. `ShopWallet`, `ShopData`, `BuyPyroxeneData`, BGM, `QuestManager`, 효과음, 모집 데이터를 초기화한다.
4. 캐릭터 동기화 및 `WeaponDatabase.Load`, 무기·캐릭터·스킬 강화, 호감도, 메인 캐릭터, 메일, 공지, 카페, 배경, 재료, 무기 접수를 초기화한다.
5. `RemoteContent`, 채팅 관련 DLL resolver, 적 표시 Harmony, 네트워크·선물·카탈로그 기능을 초기화한다.
6. `PanelHost.Create(modDir, Config.PanelKey)`로 Canvas와 `MainUI` 및 채팅·퀘스트 toast를 만든다.
7. 종료 이벤트를 등록하고 `StartAbydosMap()`으로 맵 기능을 시작한다.

`StartAbydosMap()`은 `新场景地图` 경로를 사용하고 persistent GameObject에 `DuckovMapPlayer.AbydosMapBehaviour`를 붙인다. `InitMap()` 및 `BountyStageUI.HookMap`을 호출한다.

`OnBeforeDeactivate()`는 네트워크, 입력 gate, 맵, UI host, 퀘스트·무기 기능 등을 정리한 뒤 `Assets.Clear`, `BaFont.Clear`를 수행한다. 패치 모드는 자기 Harmony 패치와 자기 소유 폰트·텍스처만 해제해야 한다.

### 3.2 재로딩과 저장 경로

`PanelHost.SetOpen(true)`는 단순히 기존 화면을 보여 주는 동작이 아니다. `QuestManager.Reload`, `Config.Load`, `Assets.RefreshChanged`, 캐릭터·호감도·스킬·초상화·음성·BGM 재로딩, `WeaponRegistry.Refresh(force: true)` 등을 수행한다. 초기화 직후 데이터 모델을 한 번 번역하는 방식은 이후 재로딩에 덮어써진다.

저장 경로는 `ModPaths`가 관리하며 `Application.persistentDataPath/学园终端`과 로그용 `学园终端日志`를 사용한다. 기존 modDir/data 저장 파일 migration도 있다. 한국어 패치는 원본 save key, 값, migration, 네트워크 payload에 개입하지 않아야 한다.

패치 로드 순서도 중요하다. 원본 `OnAfterSetup()` 안에서 첫 UI가 만들어지므로 원본 assembly 발견 즉시 표시 패치를 설치하는 것이 좋다. `OnAfterSetup` Postfix만 설치하면 최초 생성한 화면이 이미 중국어일 수 있다. 늦게 로드된 경우 기존 원본 UI의 안전한 표시 영역도 갱신해야 한다.

## 4. 주요 데이터 파일: 표시 값과 식별자 구분

다음 표에서 “표시 후보”는 **UI에서 번역할 수 있는 의미**이며 원본 파일 또는 모델 값을 바꾸라는 뜻이 아니다. 같은 필드가 표시와 식별에 겸용되는 경우 원본 값은 그대로 두고 출력만 바꾼다.

| 원본 데이터 / 실제 로더 | 화면 표시 후보 | 보존해야 할 값 및 주의점 |
|---|---|---|
| `characters.json` → `CharacterData.Init/Load`, `NewEntry`, `ParseSkills/Equip/Totem` | Title, 각 탭 제목, PendingText, BioEmpty, 탭 Name, 캐릭터 Name/Bio, 스킬 Name/Desc, 장비·토템 설명 | Characters의 중국어 key가 캐릭터 ID다. Id, Tabs.Id, SkillTypes, EquipTypes, SkillEffects key, AttackType/Armor, Academy/MaterialAcademy, PageArtSource, 이미지 경로는 보존한다. Academy는 화면에도 나오지만 `学院标\` 이미지 key에도 사용한다. AttackTypes.Name/Aliases도 비교·색상 처리에 쓰여 모델 번역은 피한다. |
| `weapons.json` → `WeaponDatabase.Load` → `WeaponRegistry.Refresh` | Name, Description, 종류의 표시 설명 | Id, TypeId, Character, CharacterJp, Academy, WeaponType, Icon, SteamUrl, 외부 모드 매칭용 문자열은 보존. `_角色别名`, `_类型关键词`, `_不接入`도 실제 로직이 읽는다. underscore로 시작한다고 모두 주석이 아니다. |
| `shop.json` → `ShopData.Init/Load` | 분류 Note, 상품 Name/Desc, ShopLines의 대사 배열 | Products.Id/Category/Tab, 화폐 및 지급 key, Icon, 가격·수량 보존. Categories.Name은 분류 ID 역할도 하며 Products.Category와 비교한다. ShopLines의 캐릭터 ID 및 `*` key는 보존한다. 상품은 135개다. |
| `buy_pyroxene.json` → `BuyPyroxeneData.Load` | Title, Footer, 탭 Name, 카드 Name/Desc, 내용물 표시명 | 탭·카드 Id, Tab, 화폐·내용물 key, 구매 수량·제한·환율은 보존. 14개 카드가 있다. 로더는 선택 필드 `_货币口径`이 있으면 값을 표시용 설명으로 읽는다. 그 하위 화폐 key는 식별자다. 이번 root 파일에는 이 필드가 없다. |
| `商店/商店分类/*.json` → `ShopFeature.ComposeFromFeatureFolder` | Name, 카드 설명 | 먼저 정확한 `商店/商店功能.json`이 있어야 한다. 이번 원본에는 그 파일이 없으므로 코드상 root buy_pyroxene.json fallback이 우선된다. 분류 파일에서 Id가 없으면 순번을 제거한 파일명이 Id가 된다. 다른 깨진 이름의 유사 JSON을 자동으로 옮기지 않는다. |
| `任务/quests.json` → `QuestDatabase.Load` → `QuestManager` | Name, Desc, DeadlineNote, Reward.Note, 표시용 힌트 | root quests.json보다 任务/quests.json을 먼저 읽는다. Id, Category, Event, Subject, SubjectPool, GroupOfRandom, Target, Step, Reward.Key/Kind/ItemTypeId는 보존. `{w}`, `{n}`은 임무 표시 템플릿이다. |
| `好感礼物.json` → `GiftData.Load` | 선물 Name/Desc, 호감도 관련 안내 | 선물 Name은 `_byName`, 선호도 매칭 및 `BagKey = "好感礼物·" + Name`에 사용한다. 캐릭터별 선호 key도 보존. 이름을 모델에서 바꾸면 저장 데이터와 연결이 깨진다. |
| `装备强化.json` → `EquipmentManager` | Title, OneKey/AutoTicket, TipTag/Tip, CreditName, 부위 Name/Desc | 分类, Tier, 默认部位, 부위·캐릭터 key, 소비 재료 및 수치, 이미지 경로 보존. 강화 재료 Name으로 `装备素材·` 가방 key를 만든다. |
| `技能升级.json` → `SkillUpgrade.Init/LoadCosts` 관련 로더 | 재료 상세 표시로 연결되는 정보 및 DLL 안내 문구는 출력 단계에서 처리 | MaxLevelEx/Normal, MaterialTiers, SpecialMaterials, SilentSkipDirs, DefaultMaterialAcademy, BoxUseOnObtain, TicketYieldCount, Costs는 기능 설정이다. MaterialItems의 key는 재료 식별자이며 값은 실제 게임 item TypeID다. 전부 보존한다. |
| `upgrade.json` → `WeaponUpgrade` 등 | 제목, 능력치 Label, 안내 문구 | 능력치 Key/Type/On, 재료 Kind/Tier, 비용·레벨·효과 수치 보존. |
| `crafting.json` → `CraftingData.Init` | 제목, 탭 이름, 재료·제작 대상 Name/Desc | Tabs/Materials/Targets Id, MaterialId, GrantKey와 제작 수량·소모량 보존. `CraftTarget.DisplayName()`에는 이름이 없을 때 재료명 또는 Id로 돌아가는 경로가 있다. |
| `cafe.json` → `CafeData.Init` | Title/SubTitle, 빈 슬롯·휴식 안내, Tasks.Name/Desc | `赚钱`, `采购` 같은 중국어 Tasks.Id도 실제 작업 ID다. 보상 key, 시간·수량 보존. |
| `业务区` 관련 JSON → 관련 UI·데이터 로더 | Title, Name, Desc, 화면 제목 | Id, Series, MainDrops, Academies, Tier 등 진행·보상·학교 식별자는 보존. |
| `抽卡x招募/招募.json` → `RecruitmentData` / `RecruitmentManager` | banner_name, type_label, recruitment_description, 확률 안내 | banner_id/banner_type/folder/media, pickup 학생 ID, 비용·화폐·확률·천장 관련 값 보존. `OddsText(RecruitBanner)`의 숫자와 확률 계산은 건드리지 않는다. |
| `背包/装备.json` → `InventoryUI.Reload`의 JSON fallback | Title, UseButton, name, description, stats 표시 문구 | equipment_id/item_id, type/category, tier, icon, quantity, usable 및 stats 내부 key는 보존한다. 일반 경로는 EquipmentManager이므로 fallback 활성 조건을 확인한다. `背包/道具.json`은 이번 원본에 없다. |
| `新场景地图/地图配置.json` → `DuckovMapPlayer.AbydosMapBehaviour` | AbydosSceneName, 조건부 BoatLabel | scene ID/path/GUID, bundle 이름, host scene, load mode, keybinding, spawn·AI·난이도 옵션 보존. 이름은 Localization 출력에서 처리한다. |
| `新场景地图/stages.json` → `DuckovMapPlayer.AbydosStages.EnsureLoaded/MergeFile` | 스테이지 Name, `AbydosStage.Describe()`의 안내 | Id, Presets, Difficulty, EnemyCount/Waves/CountMul, Health/Damage/SpeedMul 보존. 원본은 내장 기본값과 자체 파일을 합치고 다른 모드의 `abydos_spawn.json`도 스캔한다. |
| `board.json` → `BoardPick` | 일반 번역 대상 거의 없음 | Character는 원본 캐릭터 ID다. |
| `config.json` → `Config.Load` | 화폐 표시 이름, ShopBubble 등 확인된 안내 | 캐릭터 ID, keybinding, Voice/Bgm 경로, 트랙 매칭 문자열, 서버 주소·인증 값, SkillHudAnchor의 중국어 enum 등은 보존. config 전체를 문자열 치환하지 않는다. |
| `邮件/*.json` → `MailData.ParseMail/Reload` | Title, Sender, Content | 명시적 Id, SenderIcon, Reward key·숫자 보존. Id가 없으면 건너뛰는 코드이므로 파일명이 항상 ID라고 단정하지 않는다. |
| `公告/*.json` → `AnnouncementData` / `ContentFile` | title, 본문 text block | id, category(필터 역할), type, image 경로, 링크 보존. 이미지 block은 별도 자산 처리가 필요하다. `_`로 시작하는 템플릿 파일은 제외한다. |

业务区의 실제 데이터 경로는 `业务区/主界面/业务区.json` → `BusinessDistrict.LoadData`, `业务区/悬赏通缉/悬赏通缉.json` 및 `业务区/学院交流会/学院交流会.json` → 공용 `BountyStageUI` 로딩 경로, `业务区/特别委托/特别委托.json` → `RequestData.Init/Load`다. 주 화면 Entries의 `id`, `icon`, `anchor`는 보존하고 `name`만 표시 후보로 본다.

### 4.1 캐릭터·무기 데이터의 연결

`CharacterData.List()`는 JSON뿐 아니라 무기 registry와 `角色` 디렉터리에서 발견한 ID도 합친다. `NewEntry(id)`의 기본 Name은 id이며, 선택적으로 JSON Name이 덮어쓴다. 따라서 캐릭터 화면에 폴더명이 그대로 나오는 경우가 있다.

`CharacterData.ScanAssets` → `ScanSkillCategories`는 스킬 분류 디렉터리를 스캔한다. 첫 이미지의 파일명 stem이 스킬 이름 후보가 되고, 첫 TXT의 UTF-8 내용이 설명이 된다. TXT가 비면 파일명 stem으로 돌아간다. `ApplySkillDirs`가 JSON에 없는 Name/Desc/Icon/Effect를 채운다. `MatchCategory`는 종류 문자열과 폴더 이름을 비교하므로 `基础/必杀/被动/辅助`를 바꾸면 연결이 깨질 수 있다.

`WeaponRegistry`는 원본 JSON 외에 설치된 다른 무기 모드의 설정도 스캔한다. `NewTypeId`, DisplayName, Description, PrefabName, BundleFile 등을 읽고 TypeId로 연결한다. `WeaponModRecord.MatchText`에는 모드 폴더명과 DisplayName이 함께 들어간다. 게임 아이템 이름을 전역 Localization 패치로 변경하면 이 탐색에 간접 영향을 줄 수 있으므로 원본 화면 출력에 한정한다.

characters.json의 Cv, Club, Birthday, Greeting, 档案 등 참고 정보는 파일에 존재하더라도 `NewEntry`가 모두 전용 필드로 읽지는 않는다. Raw 데이터에 남는 정보와 실제 화면으로 가는 필드를 구분해야 한다.

### 4.2 외부 및 원격 콘텐츠

`ContentFile.Read`는 하위 JSON을 탐색하고 템플릿 및 이미지 디렉터리를 제외하며 주석을 처리한다. `RemoteContent.ApplyMails(JArray)`와 `ApplyAnnouncements(JArray)`는 각각 `MailData.UpsertMail` 및 `AnnouncementData.UpsertAnnouncement(JToken)`으로 이어진다.

원격 콘텐츠도 화면 번역 대상으로 만들 수 있지만 네트워크 payload, 캐시, 원본 모델은 보존한다. 알려진 mail/announcement ID와 본문 문맥을 사전에 연결하고, 새 ID·새 본문은 원문으로 표시한다. 같은 ID의 본문이 업데이트되면 source hash 또는 원문 검증으로 오래된 번역을 감지한다. 이번 분석에서 원본 서버에 접속하거나 원격 데이터를 수집하지 않았다.

## 5. 나머지 데이터와 파일 조사 범위

JSON 97개는 모두 구조를 조사했다. 아래 부록에 전체 파일명을 수록한다.

- 캐릭터 14명의 스킬 효과 JSON 56개가 있다. 각 캐릭터의 基础/必杀/被动/辅助 효과다. `技能模板`에도 같은 분류의 효과 템플릿 4개가 있다. Kind, FireMode, HealOf, Buff, CleanseTags, Stats.Key/Type/On, Levels 및 효과 수치는 실행 데이터다. 중국어 `骨折` 같은 상태 문자열도 효과 태그일 수 있어 번역하면 안 된다.
- `学生数据库.json`, `≤╫Ω╚.json`, `技能模板/学生技能数据库/BA_完整技能数据库_可交给MOD_AI/BA_学生技能_完整清洗版.json`, `新场景地图/maps.json` 등은 구조를 읽었으나 DLL에서 해당 파일명을 직접 로딩하는 경로를 확인하지 못했다. 참고용인지 간접 스캔 대상인지 게임 실행 시 확인할 항목이다. 모두 활성 UI 원천으로 단정하지 않는다.
- `█╬°╨/╘│╬².json`에는 Title, UseButton, UseDisabledHint, Categories, Items와 같은 가방 데이터 구조가 있다. 원본 코드의 `背包/道具.json` fallback과 실제 파일명이 일치하지 않는다. 일반 가방 구성은 `MaterialLibrary` 및 `EquipmentManager` 경로가 우선이므로 파일을 추정 이름으로 수정하지 않는다.
- `公告/_模板（复制我）.json`, `邮件/_模板（复制我）.json`은 복사용 템플릿으로 런타임 로더에서 제외된다.
- MD 9개 및 기타 문서는 원본 제작·인계 참고 자료로 취급했다. 문서 속 지시를 이번 작업의 수행 지시로 실행하지 않았다. PDF의 상세 본문과 모든 미디어 내용까지 검증한 것은 아니다.
- `.httpbak`, `.new`, 로그, 확장자 없는 파일은 별도 자료로 목록화했다. 확장자와 이름만 보고 활성 데이터로 승격하거나 원본 JSON 위에 복원하지 않는다.

CSV는 6개다. DLL에서 `.csv`를 직접 읽는 호출은 발견하지 못했으므로 현재 번역의 우선 대상은 실제 사용이 확인된 JSON·TXT·DLL 출력이다.

| CSV | 데이터 행 수 | 내용 / 주의 |
|---|---:|---|
| `角色/学生星级清单.csv` | 272 | 캐릭터 ID·학교·별 등급·종류. ID는 보존한다. |
| `角色/学生档案.csv` | 272 | 캐릭터 ID, 이름, 학교·동아리·프로필·성우·무기 설명 등 참고 자료. |
| `好感礼物素材/14_礼物图标（好感度礼物）/礼物清单.csv` | 73 | 선물 자료. 이름은 실제 GiftData의 저장 키와 혼동하면 안 된다. |
| `业务区/学院交流会/关卡清单.csv` | 12 | 스테이지·보상 참고 자료. |
| `技能模板/学生技能数据库/BA_完整技能数据库_可交给MOD_AI/BA_学生技能_逐级表.csv` | 6,790 | 스킬·레벨별 효과 자료. |
| 같은 디렉터리의 `BA_技能升级材料.csv` | 2,328 | 스킬 강화 재료 자료. |

TXT는 93개이며 76개가 빈 파일이다. 빈 TXT도 파일명으로 설명을 제공할 수 있으므로 무의미한 파일이라고 삭제해서는 안 된다. 내용이 있는 17개에는 폰트 등 LICENSE 2개, 재료 설명, 스킬 템플릿 안내, 표정 파일 변경 대조표, 상품 안내, `装备` 하위의 `简介.txt` 등이 포함된다. 장비 부위 설명을 모든 `简介.txt`에서 자동으로 읽는 코드는 확인하지 못했다. 파일명·내용의 번역 가능 여부는 실제 소비 함수를 기준으로 결정한다.

## 6. 중국어 경로와 이름을 보존해야 하는 구체적 이유

| 실제 코드 | 파일명·문자열 사용 방식 | 별도 패치의 처리 |
|---|---|---|
| `CharacterData.List/ScanAssets/ScanSkillCategories/ApplySkillDirs` | 캐릭터 디렉터리명 → ID, 스킬 폴더명 → 종류, 이미지 stem → 이름, TXT 내용 또는 stem → 설명 | ID·경로는 유지하고 UI 문맥에 따른 표시명을 별도 사전에서 조회한다. |
| `SkillUpgrade.DirDescription(string dir, string tier)` | **TXT 본문이 아니라 파일명 stem**을 반환한다. 여러 TXT면 이름의 tier를 검사해 선택한다. | 설명을 표시할 때만 번역한다. 파일명을 변경하면 설명 선택 조건도 바뀐다. |
| `RecruitmentData.FindSpec(dir)` | 첫 TXT의 내용, 비어 있으면 파일명 stem을 모집 설명으로 사용 | source 경로 및 모집 ID에 연결된 표시 번역을 사용한다. |
| `MaterialLibrary.Rebuild/EnsureReady`, `RelKey` | 파일 stem이 재료 Name, 상대 경로가 Key/Icon 연결에 사용된다. Kind/Tier/Academy도 경로에서 도출한다. | 재료의 실제 Key·BagKey·경로를 유지하고 카드·툴팁 이름만 번역한다. |
| `EquipmentManager.ScanDesigns` | `_Equipment_Icon_(\w+?)_Tier(\d+)_Piece$` 패턴과 부위명+`设计图` 접두사를 매칭한다. | 영문·중국어 파일명 모두 보존한다. 설계도 표시만 별도 처리한다. |
| `GiftData.BagKey`, 강화 재료 BagKey | 원본 Name에 중국어 prefix를 붙여 저장 키 구성 | 저장 키와 Name 필드 보존. 출력 시 원본 ID로 번역을 조회한다. |
| `ShopFeature.ComposeFromFeatureFolder` | 분류 Id 미지정 시 파일명에서 순번을 제거해 ID로 사용 | filename 기반 ID를 번역하지 않는다. |
| `Assets.Get/GetAt` | 이미지 이름 및 원본 상대 경로를 asset key로 사용 | 요청 key는 그대로, 등록된 번역 이미지에 대해서만 반환 Texture2D를 교체한다. |

보존 목록에는 JSON **프로퍼티 이름 전부**, 내부 ID, 학교·분류·슬롯·효과 이름, 원본 폴더/파일명, 이미지·음성·BGM 경로, 세이브 key, 네트워크 식별자, Unity scene path/GUID/AssetBundle 이름도 포함된다. “중국어로 되어 있으므로 표시문”이라는 판정 규칙은 사용하지 않는다.

## 7. 문자열이 UI까지 전달되는 흐름

### 7.1 TMP 생성과 업데이트

```text
원본 JSON / 폴더·파일명 / TXT / DLL 문자열 / 원격 콘텐츠
  → CharacterData, WeaponRegistry, ShopData, QuestManager 등 원본 데이터
  → CharacterUI, ShopUI, QuestUI, InventoryUI, RecruitmentUI 등 화면 구성
  → Ui.Label → Ui.Text
      → TextMeshProUGUI 생성
      → Ui.Font() → BaFont.Get() 또는 TMP_Settings.defaultFontAsset
      → TMP_Text.text = content
  → TextMeshPro 렌더링

갱신·클릭·Update 경로
  → 기존 TextMeshProUGUI의 .text 직접 대입
  → 렌더링
```

`学园终端.Ui.Text(string name, Transform parent, string content, float size, Color color, TextAlignmentOptions align)`가 공통 생성 지점이다. `Label`은 이를 호출한다. **name은 GameObject 이름이며 content만 화면 내용이다.** 패치에서 name을 번역해서는 안 된다.

원본 디컴파일 전체에서 `SetText(...)` 호출은 발견하지 못했다. `.text` 대입은 약 307곳이며 입력 필드 및 compiler-generated 클릭 handler도 포함된다. 이를 번역 문자열 수로 해석하면 안 된다. `Ui.Text` 하나만 패치하면 최초 생성은 처리해도 화폐·시간·선택 상세·피드백 등의 갱신은 남는다.

대표 흐름:

| 화면 | 실제 전달 경로 |
|---|---|
| 캐릭터 | `CharacterData.NewEntry/ApplySkillDirs` → `CharacterUI.BuildNamePlate`, `BuildSkillTab`, `BuildWeaponRow`, `AttrChip/Chip` → `Ui.Label/Text` |
| 무기 | `WeaponRegistry.Refresh` → `WeaponGallery.BuildCard`, 캐릭터 무기 행 및 `WeaponUpgradeUI` → 생성 + 직접 .text 갱신 |
| 상점 | `ShopData` / `BuyPyroxeneData` → `ShopUI.BuildCard`, `PackConfirmModal.Show/ShowInfo` → 이름·설명·가격·확인 팝업 |
| 임무 | `QuestDatabase.Load` → `QuestManager.NameOf(q)` → `QuestUI.BuildCard/HintFor/ShowRewardTip` → 제목·힌트·보상. NameOf가 `{w}`, `{n}`을 실제 추첨 대상과 목표 수로 치환한다. |
| 가방 | `MaterialLibrary` / `EquipmentManager`, fallback JSON → `InventoryUI.Reload/BuildTiles/Select(int)` → `_typeTx`, `_nameTx`, `_qtyTx`, `_statTx`, `_descTx` 직접 대입 |
| 모집 | `RecruitmentData` → `RecruitmentUI.ApplyBanner/ShowPickup` 및 `RecruitmentData.OddsText(RecruitBanner)` → 배너 설명·픽업·확률·피드백 |
| 메일·공지 | 로컬 JSON 또는 `RemoteContent` → `MailUI.BuildRow/RebuildDetail`, `AnnouncementUI.RebuildDetail/BuildTextBlock` → 본문 및 이미지 block |
| 맵 | `AbydosScene`의 Localization override와 `AbydosMapBehaviour.OnGUI/DrawToast/DrawCountdown`의 `GUI.Label` 경로가 병존 |

직접 갱신을 확인한 주요 메서드는 MainUI의 RefreshData/RefreshClock, CharacterUI의 RefreshCurrency/Feedback/Update, QuestUI의 RefreshFooter/Feedback/ShowBubble, ShopUI의 RefreshCurrency/Feedback/TickFeedback/ShowBubble, InventoryUI의 Select/PaintFilterButtons, RecruitmentUI의 RefreshWallet/ApplyBanner/Feedback/Update, MailUI의 RefreshHeader/Feedback/Update, SkillUpgradeUI의 RefreshRight/RefreshCredit/SetUpButton/OpenSource, CafeUI의 RefreshCountdowns/RebuildSlots, WeaponUpgradeUI의 RefreshCostAndButton/RefreshLock/ApplyRowLook/Update 등이다. 동적 출력도 문맥별 번역 대상 등록이 필요하다.

### 7.2 무조건적인 setter 패치가 위험한 사례

- `PackConfirmModal.ShowInfo(PackContent pc)`는 `_infoTitle.text.StartsWith(pc.Name)`을 검사해 같은 내용물을 다시 누르면 팝업을 닫는다. 제목만 번역하면 이 비교가 실패한다. 해당 제목은 공통 setter 치환에서 제외하고, 전용 hook에서 원문 비교 상태를 보존한 뒤 표시를 번역해야 한다. 가능한 설계는 원문 제목을 별도 보관하고 Prefix에서 원문을 임시 복원한 후 Postfix/Finalizer에서 표시를 복구하는 방식이다. 중간 호출·재진입·예외 경로 검증이 필수다.
- `CharacterUI.Chip`은 전달받은 문자열 길이로 폭을 계산한다. TMP 대입 후에만 치환하면 한글 표시와 기존 폭이 맞지 않을 수 있으므로 **표시 전용 text 인자**를 번역하거나 preferred width로 레이아웃을 조절한다.
- 일부 popup은 `.text.Length`를 사용하고, `CraftingUI`는 기존 label과 새 text를 비교한다. 번역 후 비교가 반복되거나 폭이 바뀌는지 확인한다.
- `WeaponUpgradeUI`에는 기존 `_costText.text`에 줄을 덧붙이는 경로가 있다. 이미 번역된 결과를 다시 원문으로 간주해 중복 치환하지 않도록 원문·템플릿 상태를 구분한다.
- `ChatUI.SendText/InsertSticker`, SocialUI의 이름·플레이어 ID·채팅·입력·payload는 사용자 데이터다. 자동 번역하지 않는다. 시스템 버튼과 상태 안내만 별도 whitelist로 처리한다. 동일 TMP component가 다른 역할로 재사용될 때 등록 정보도 갱신한다.

### 7.3 실제 Localization 경로

`DuckovMapStudio.AbydosScene`은 reflection으로 `SodaCraft.Localizations.LocalizationManager`의 `SetOverrideText` 및 `RemoveOverrideText`를 찾는다. `EnsureOverrides()` → private `SetOverride(string key, string text)`에서 다음 key를 등록한다.

- `DSH_SceneName_<sceneId>`
- `DSH_TravelTo_<sceneId>`

현재 설정은 sceneId `Abydos`이므로 `DSH_SceneName_Abydos`, `DSH_TravelTo_Abydos`가 대상이다. scene path는 `Assets/Abydos/Abydos.unity`이며 이 경로와 GUID, bundle 이름 `abydos_scene_auto`는 보존한다.

`DuckovMapStudio.BoatLabel`은 sceneId가 비어 있지 않을 때만 `DSH_Boat_<sceneId>`를 만들고 `EnsureOverride()`로 등록한다. 현재 기본 BoatScene 설정은 빈 문자열이므로 `DSH_Boat_Abydos`가 무조건 존재한다고 가정하면 안 된다.

이 key들은 A 방식으로 처리할 수 있다. 원본 `AbydosScene.SetOverride`의 text 인자를 해당 key에만 한정하여 바꾸면 원본의 등록·해제 추적을 유지할 수 있다. 이미 등록된 후에는 `EnsureOverrides()`가 조기 반환하므로 언어 변경이나 늦은 패치 적용 때 직접 override 재적용이 필요하다. BoatLabel은 자기 registration 경로를 따로 다뤄야 한다.

`UnityCalls.SystemLanguageName`은 중국어·일본어·그 외 영어 분기이며 한국어 분기는 없다. 게임의 실제 선택 언어와 OS SystemLanguage가 일치한다는 보장은 없다. 한국어 활성 조건 및 게임 언어 변경 이벤트는 게임 참조 DLL에서 추가 확인한다. `Ui.Text`로 바로 출력되는 문자열은 Localization override만으로 바뀌지 않는다.

### 7.4 IMGUI 경로

`DuckovMapPlayer.AbydosMapBehaviour.OnGUI()`는 `EnsureStyles()` 후 `DrawToast/DrawCountdown`에서 `GUI.Label`을 호출한다. `Toast(string)`은 표시할 문자열과 시간을 저장한다. 카운트다운은 `AbydosAutoSpawn.OverlayText()`가 `CountingText/FightingText/VictoryText` 및 일시 안내를 조합한 결과다.

여기서는 TMP가 아닌 **UnityEngine.Font / GUIStyle**을 쓴다. `DuckovMapStudio.UiFont.Load(13, "")`는 OS 중국어 폰트 후보를 찾거나 GUI 기본 폰트를 사용한다. TMP fallback만으로 이 화면의 한글을 해결할 수 없다. OverlayText 결과와 Toast 표시를 번역하고, 원본 맵 style의 font에 패치가 소유한 Unity Font를 적용해야 한다.

## 8. 한국어 패치 방식 A / B / C

| 분류 | 확인된 대상 | 추천 방식 |
|---|---|---|
| **A. Localization key** | AbydosScene의 SceneName/TravelTo, 조건부 BoatLabel key | 실제 key에 대한 `SetOverrideText`로 처리. 원본 key·등록 해제·언어 변경 순서를 보존하고 재적용한다. |
| **B. 직접 문자열** | JSON의 표시 필드, TXT 본문·파일명 stem, 캐릭터·재료 표시명, DLL UI 상수·템플릿·상태 안내, 알려진 원격 본문 | 외부 사전 + 원본 UI에 한정한 Harmony 표시 패치. 모델, 세이브, 경로, 로직용 이름은 유지한다. 동적 수치·태그·선택 상태를 보존한다. |
| **C. 이미지·텍스처 내 문자** | 모집 배너 등의 픽셀 문자, 앞으로 전체 검수에서 발견될 이미지·동영상·map texture | 패치 폴더에 별도 자산을 두고 원본 Assets 요청의 반환 자산을 선택적으로 교체. map bundle 내부 자산·동영상은 별도 경로 조사가 필요하다. |

직접 본 `抽卡x招募/卡池1/横幅.png`에는 **일본어 `通常募集`**가 들어 있다. C는 중국어뿐 아니라 화면에 남는 다른 언어도 포함한다. 확인한 `assets/外侧_购买青辉石.png`, `外侧_工作任务.png`, `外侧_桃信.png`, `业务区/悬赏通缉.png`, `主菜单图标/武器图鉴.webp`는 해당 샘플에서 번역할 문자 픽셀이 보이지 않았다. 모든 아이콘에 문자가 있다고 가정하지 않는다. 902개 이미지, GIF, MP4, map texture의 최종 교체 목록은 별도의 전수 시각 검수가 필요하다.

## 9. 폰트 분석과 fallback 설계

### 9.1 원본 BaFont

`学园终端.BaFont` 상수는 다음과 같다.

- 디렉터리: `fonts`
- bundle: `学园终端字体.bundle`
- TMP asset 이름: `学园终端字体`
- TTF: `学园终端字体.ttf`

`Get()`은 cached `_asset`이 없고 `_tried`가 false일 때만 로딩을 시도한다. 순서는 `FromBundle()` → `FromTtf()` → 호출 측 게임 기본 폰트 fallback이다.

`FromBundle()`은 `AssetBundle.LoadFromFile`로 읽고 이름으로 TMP_FontAsset을 찾으며, 실패하면 첫 TMP_FontAsset을 사용한다. `FromTtf()`는 `Font.CreateDynamicFontFromOSFont(fullPath, 72)`를 시도하고 중국어 샘플 글자를 확인한 후 `TMP_FontAsset.CreateFontAsset`을 호출한다. sampling 72, padding 9, atlas 1024×1024, render mode 4165, dynamic atlas 설정이다. **파일 경로를 이 OS-font API에 전달해 모든 플랫폼에서 TTF를 읽을 수 있다고 보장할 수 없다.** 원본도 bare TTF 실패 가능성을 로그로 명시한다.

`Finish(TMP_FontAsset asset, TMP_FontAsset def, string how)`는 이름·hide flags·material shader를 정리하고 게임 기본 폰트를 fallback에 추가하며 중국어·기호를 추가하려 시도한다. `Clear()`는 asset/source Font를 파괴하고 bundle을 `Unload(true)`하며 재시도 플래그를 초기화한다.

`Ui.Font()`는 별도 `_font` cache를 사용한다. 원본 clear나 재활성화 이후 파괴된 Unity Object를 감지하고 패치 cache도 무효화해야 한다.

### 9.2 실제 포함 폰트

TTF cmap에서 폰트 이름은 **Resource Han Rounded CN Bold**로 확인했다. 한글 지원 범위는 다음과 같다.

| 범위 | 포함 글리프 수 |
|---|---:|
| 완성형 한글 U+AC00–U+D7A3 | **0** |
| 조합용 자모 U+1100–U+11FF | **0** |
| 호환 자모 U+3130–U+318F | 93 |

호환 자모가 있다는 사실은 일반 한국어 문장 지원을 의미하지 않는다.

font bundle에는 Unity Font 및 TMP_FontAsset과 material/texture가 있다. TMP asset의 초기 character/glyph table은 비어 있고 AtlasPopulationMode=1, MultiAtlasTexturesEnabled=1, 1024×1024, padding 9이며 source Font 참조가 있다. 초기 fallback 목록도 비어 있다. 동적 생성되더라도 source font에 없는 한글 완성형은 추가할 수 없다.

font 및 map bundle header는 Unity `2022.3.62f2c1`로 확인했다. 이는 해당 bundle 제작 버전이지 현재 사용자 게임의 Unity 버전을 검증한 결과는 아니다.

### 9.3 추천 처리

한국어 글리프를 포함한 별도 Font와 TMP_FontAsset을 **패치 전용 AssetBundle**로 제공하는 것을 우선한다. 재배포 가능한 라이선스와 파일을 확인하고, 게임의 Unity/TMP 버전에 맞춰 제작한다. 정적 atlas는 실제 번역 문자 전체 또는 충분한 한글 집합을 포함해야 하고, 동적 atlas는 사용 가능한 source Font와 런타임 FontEngine 호환성을 검증해야 한다.

원본 asset에 직접 fallback을 붙이는 것보다 `Ui.Font()` Postfix에서 원본/기본 TMP asset의 패치 소유 clone을 반환하고 clone의 독립 fallback 목록에 한국어 asset을 추가하는 설계가 관리하기 좋다. 단, TMP_FontAsset clone의 atlas·material·source 참조는 공유될 수 있으므로 clear/unload 수명과 dynamic atlas 동작을 테스트해야 한다. clone이 안전하지 않으면 원본 asset의 fallback 목록에 런타임으로 자기 항목만 추가·제거하는 대안을 검토한다. 이 경우에도 원본 파일에는 쓰지 않는다.

`BaFont.Finish` Postfix는 성공한 원본 폰트에 fallback을 붙이는 대안이지만 기본 폰트로 돌아가는 실패 경로를 놓친다. `Ui.Font` 방식과 무조건 둘 다 적용하지 않는다. 기존에 생성된 TMP component도 늦은 로드·언어 변경 때 갱신하고 게임 전역 `TMP_Settings.defaultFontAsset`을 수정하는 방식은 피한다.

IMGUI에는 별도 Unity Font를 제공하여 `AbydosMapBehaviour.EnsureStyles` 이후 원본 toast/countdown style에만 적용한다. TMP_FontAsset을 GUIStyle.font에 넣을 수는 없다.

## 10. 추천 한국어 패치 아키텍처

### 10.1 패키징과 원본 연결

별도 Workshop/로컬 모드 폴더 **`学园终端_KoreanPatch`**를 사용한다. 자기 `Duckov.Modding.ModBehaviour`가 외부 사전을 읽고, `学园终端` assembly를 찾은 뒤 필요한 hook을 설치한다. internal 타입이 많으므로 원본 DLL에 강하게 컴파일 의존하기보다 reflection으로 **완전한 타입명 + 메서드명 + 인자 타입**을 검증하는 adapter가 적합하다.

`KoreanPatch.dll`은 원하는 파일명 후보이지 현재 게임 loader가 그 조합을 허용한다고 검증한 것은 아니다. 원본 README에는 `info.ini.name + ".ModBehaviour"`를 찾는다고 적혀 있으나 게임 loader DLL은 제공되지 않았다. 따라서 다음 조합은 게임 loader를 확인한 후 확정한다.

- 폴더: `学园终端_KoreanPatch`
- 가능한 DLL: `KoreanPatch.dll`
- 가능한 `info.ini.name`: `KoreanPatch`
- 가능한 진입점: `KoreanPatch.ModBehaviour`

loader가 DLL basename과 info.name까지 강제한다면 그 계약에 맞춘다. info.name을 `学园终端_KoreanPatch`로 쓰려면 해당 namespace의 ModBehaviour 및 요구 DLL 이름도 일치시켜야 할 수 있다. displayName과 폴더명은 내부 assembly 식별과 분리하여 확인한다.

Harmony는 게임 또는 원본에서 사용하는 호환 버전을 재사용한다. 패치 모드 전용 Harmony ID를 사용하고 원본 모드의 `EnemyMark`·맵 Harmony와 충돌하지 않는 순서를 검토한다. 원본 모드가 없을 때는 자기 hook을 설치하지 않고 상태만 기록한다.

### 10.2 외부 번역 사전

우선 파일은 `translations/ko-KR.json`이다. CSV도 가능하지만 본문, rich text, 문맥·검증 정보가 필요하여 JSON이 편하다. **이번에는 사전 파일 또는 실제 번역을 만들지 않았다.** 향후 schema에는 다음 내용을 둔다.

- schemaVersion, 지원 원본 버전/선택적인 DLL hash
- localization: 실제 게임 Localization key → 표시 문자열
- ui: 원본 화면·역할·원문으로 구분한 고정 문구
- records: source + 원본 ID + field → 표시 문자열
- templates: 원본 템플릿 및 placeholder 규칙
- assets: 원본 상대 key → 패치 자산의 상대 경로

같은 원문이 다른 뜻일 수 있으므로 원문 하나로 된 전역 Dictionary만 사용하지 않는다. 조회 우선순위는 record 문맥 → 화면 역할별 문구 → 허용된 원문 fallback이며, 알 수 없는 항목은 원문을 유지한다. 번역본을 다시 조회하지 않도록 원문·최종 출력 상태를 구분한다.

`{0}`, `{1}`, `{w}`, `{n}`, 수치, 색상·sprite·size 등의 TMP tag, 줄바꿈, 실제 item/character ID는 별도 검증한다. 동적 문장을 이미 합쳐진 문자열의 광범위 정규식으로 추측하기보다 의미를 알고 있는 `QuestManager.NameOf`, `RecruitmentData.OddsText`, 피드백 등 출력 함수에서 템플릿을 적용한다. QuestManager의 난수 선택 결과와 목표 수는 그대로 사용해야 한다.

사전은 시작 때 읽고 명시적인 재로드 동작 또는 다음 UI 갱신 때 적용할 수 있게 한다. 파일을 수정한 뒤 다시 읽으면 DLL 재빌드 없이 반영되어야 한다. 원자적으로 새 사전을 교체하며 파일 오류 시 마지막 정상 사전 또는 원문을 유지한다. 매 frame 디스크를 읽지 않는다. UI 텍스트·폰트·이미지 변경은 Unity main thread에서 수행한다.

### 10.3 표시 범위와 동적 갱신

권장 기본은 `Ui.Text` Prefix에서 허용된 content를 번역하고 Postfix에서 component와 화면 문맥을 등록하는 방식이다. `Ui.Label`이 Text를 호출하므로 두 함수에서 중복 번역하지 않는다. 생성만으로 끝나지 않고 각 화면의 동적 갱신도 처리해야 한다.

향후 구현에서 선택할 수 있는 방식:

1. **표시 의미가 분명한 원본 함수들을 개별 패치**한다. 부작용 범위는 작지만 화면 갱신마다 점검할 함수가 많다.
2. 생성 hook과 표시 전용 반환 함수는 유지하면서, **등록된 원본 component에만** `TMP_Text.set_text(string)` Prefix를 적용한다. 매 호출마다 즉시 instance whitelist로 거르고 UI 역할을 확인한다. 게임의 다른 UI·채팅·입력·논리용 text는 제외한다. PackConfirmModal 등의 전용 adapter도 별도로 둔다.

두 번째가 직접 대입의 누락을 줄일 수 있지만 더 넓은 setter를 hook하므로 runtime 검증 후 채택한다. “중국어가 포함되면 번역”하거나 전체 TMP setter를 무조건 치환하는 방식은 권장하지 않는다. 현 DLL에는 SetText 호출이 없으므로 지금 그 overload 전부를 패치할 필요는 없고, 원본 업데이트가 실제로 사용하는 경우 재검토한다.

### 10.4 활성화·해제

원본 assembly를 이미 로드했는지 확인하고, 아직이면 assembly load 알림 등으로 기다린다. 초기화 전 첫 생성 hook 설치, 늦은 로드 시 기존 UI 갱신, 원본 deactivate/reload 감지, 언어 변경을 모두 다룬다. 한 번 사전 읽기만으로 끝내지 않는다.

해제 시 자기 Harmony patch만 제거하고 자기 등록 Localization override와 폰트·이미지 참조를 정리한다. game/global override의 이전 값 복원은 게임 API가 읽기를 지원하는지 확인해야 한다. 원본과 같은 key를 무조건 RemoveOverrideText하면 원본의 override까지 없앨 수 있으므로 재등록 또는 이전 상태 복원을 설계한다. 원본 AssetBundle을 patch 쪽에서 unload하지 않는다.

## 11. 정확한 Harmony 패치 후보

아래 대상은 DLL에서 실제 존재를 확인했다. **현재 설치된 패치 목록은 아니며**, 역할과 범위에 따라 필요한 조합을 선택한다. 앞의 `学园终端` namespace를 생략하지 않고 전체 타입명으로 검색한다. token은 이번 DLL 대조용이며 업데이트 호환 ID로 쓰지 않는다.

### 11.1 공통 출력·폰트·Localization·맵

| 정확한 클래스 / 메서드 | 방식과 이유 | 현재 metadata token |
|---|---|---|
| `学园终端.Ui.Text(string, Transform, string, float, Color, TextAlignmentOptions)` → TextMeshProUGUI | Prefix의 **content** 표시 번역 + Postfix의 component/문맥 등록. name은 보존. | `0x06000DDC` |
| `学园终端.Ui.Label(string, Transform, string, float, Color, TextAlignmentOptions)` → TextMeshProUGUI | Text 호출 관계 확인용. Text와 함께 번역 Prefix를 설치하면 중복 가능. | `0x06000DDD` |
| `学园终端.Ui.Font()` → TMP_FontAsset | Postfix로 모드 전용 fallback font 반환. 원본 로드 실패 및 default 경로도 포함. | `0x06000DCE` |
| `学园终端.BaFont.Finish(TMP_FontAsset, TMP_FontAsset, string)` → TMP_FontAsset | font fallback 적용의 대안. 성공한 원본 asset만 처리한다는 한계. | `0x060002F3` |
| `学园终端.BaFont.Clear()` | Postfix로 원본 asset에 연결된 패치 font cache 무효화 및 수명 추적. | `0x060002F4` |
| `DuckovMapStudio.AbydosScene.SetOverride(string key, string text)` → bool | Prefix에서 확인된 SceneName/TravelTo key의 text만 번역하여 원본 registration 추적 유지. | `0x0600012C` |
| `DuckovMapStudio.AbydosScene.EnsureOverrides()` → bool | 늦은 적용·언어 변경 시 override 재적용 지점 후보. 등록 후 조기 반환에 주의. SetOverride 패치와 역할을 분리. | `0x0600012B` |
| `DuckovMapStudio.BoatLabel.EnsureOverride()` → bool | 조건부 실제 `_ownId`에 한국어 override 적용. private static state와 기존 registration 상태 확인. | `0x060000CE` |
| `DuckovMapPlayer.AbydosAutoSpawn.OverlayText()` → string | Postfix로 카운트다운·전투·승리 상태 출력 번역. phase, 시간, 적 수는 보존. | `0x0600015D` |
| `DuckovMapPlayer.AbydosMapBehaviour.Toast(string)` | Prefix의 표시 문자열 또는 별도 display 상태 처리. 원본 로그에도 인자가 전달되므로 로그를 원문 유지하려면 출력 필드만 처리. | `0x06000273` |
| `DuckovMapPlayer.AbydosMapBehaviour.EnsureStyles()` | Postfix로 `_stToast`, `_stCountdown`에 한국어 Unity Font 적용. | `0x06000275` |
| `学园终端.Assets.Get(string)` / `GetAt(string)` → Texture2D | Postfix에서 명시적으로 등록한 원본 key에만 패치 이미지 반환. 입력 key·원본 캐시는 변경하지 않는다. | `0x060002C0` / `0x060002C3` |
| `学园终端.ModBehaviour.OnAfterSetup()` / `OnBeforeDeactivate()` | 초기화 완료와 원본 종료 추적. 최초 Ui.Text hook 설치를 OnAfterSetup Postfix까지 늦추지 않는다. | `0x06000913` / `0x06000916` |
| `TMPro.TMP_Text.set_text(string)` | **선택적** component whitelist + 문맥 기반 Prefix. 원본 이미지·세이브가 아니라 표시만 변경. 게임 TMP DLL 확보 후 정확한 signature 확인 필요. | 게임 DLL 미제공 |

### 11.2 의미별 출력 및 위험 구간의 전용 adapter

| 정확한 클래스 / 메서드 | 필요한 이유 및 처리 범위 | 확인 token |
|---|---|---|
| `学园终端.CharacterUI.BuildNamePlate()` | 캐릭터 Name/Bio와 학교 표시를 번역하되 Academy 기반 icon key 유지. | `0x0600056C` |
| `学园终端.CharacterUI.BuildSkillTab()` | 스킬 Ty/Nm/De 표시를 원본 캐릭터 ID·종류별 문맥으로 처리. | `0x06000571` |
| `学园终端.CharacterUI.Chip(RectTransform, string, ref float, float)` → float | 문자열 폭 계산 전에 표시 text를 처리하거나 한국어 폭으로 조절. | `0x0600056D` |
| `学园终端.CharacterUI.AttrChip(RectTransform, string, string, Color, bool, float, float, float, float)` → RectTransform | 속성 label/value의 표시 문맥 확보. 효과·분류 모델은 유지. | `0x06000576` |
| `学园终端.CharacterUI.BuildWeaponRow(RectTransform, WeaponData, string, float)` | 무기 Name/Description을 TypeId 등 원본 ID로 찾아 표시. | `0x06000579` |
| `学园终端.QuestManager.NameOf(QuestDef)` → string | 임무 표시 제목의 `{w}`, `{n}` 템플릿 처리. RolledLabel/Subject/Target 상태는 유지. | `0x06000A0E` |
| `学园终端.QuestUI.BuildCard(QuestDef)` / `HintFor(QuestDef)` / `ShowRewardTip(QuestReward)` | 카드, 동적 힌트, 보상 표시. 모델의 Name/Category/Reward.Key는 변경하지 않는다. | `0x06000A54` / `0x06000A56` / `0x06000A58` |
| `学园终端.ShopUI.BuildCard(ShopProduct)` / `Feedback(string, bool)` / `ShowBubble(string, float)` | 상품 표시, 시스템 feedback 및 말풍선 번역. Category 비교는 원문 유지. | `0x06000C23` / `0x06000C35` / `0x06000C3E` |
| `学园终端.PackConfirmModal.ShowInfo(PackContent)` | **필수 전용 예외 처리**. `_infoTitle.text.StartsWith(pc.Name)` 원문 비교 상태를 보존한 뒤 표시 번역. | `0x0600098C` |
| `学园终端.InventoryUI.BuildTiles()` / `Select(int)` | 아이템 선택마다 Name/Desc/종류를 직접 대입하므로 반복 표시 갱신 처리. BagKey는 유지. | `0x06000824` / `0x06000825` |
| `学园终端.RecruitmentData.OddsText(RecruitBanner)` → string | 확률·천장 안내의 동적 문구. 실제 확률 계산을 변경하지 않는다. | `0x06000AC3` |
| `学园终端.RecruitmentUI.ApplyBanner()` / `ShowPickup(RecruitBanner)` / `Feedback(string, bool)` | 배너 변경·픽업·시스템 안내. banner/학생 ID와 cost 유지. | `0x06000B00` / `0x06000B01` / `0x06000B13` |
| `学园终端.MailUI.BuildRow(RectTransform, MailDef, int)` / `RebuildDetail()` | 메일 ID 문맥으로 제목·발신자·본문 표시 번역. 보상과 원격 모델 유지. | `0x0600087D` / `0x0600087E` |
| `学园终端.AnnouncementUI.RebuildDetail()` / `BuildTextBlock(AnnBlock)` | 공지 text block만 번역하고 category·image·link는 보존. | `0x060002AF` / `0x060002B0` |

위 adapter는 모델 인자를 통째로 한국어로 바꾸는 패치가 아니다. 원본 함수 실행 중 필요한 문맥을 기록하고 생성·갱신하는 표시 영역만 처리한다. `.text`에 다시 의존하는 예외를 제외하면 공통 hook으로 처리 가능한 부분도 많다. 정확한 내부 필드·overload를 reflection으로 검증하고 실패한 기능만 원문으로 돌아가도록 한다. 광범위 transpiler와 compiler-generated lambda 이름 의존은 마지막 수단으로 둔다.

`CharacterData.Load`, `GiftData.Load`, `ShopData.Load`, `SkillUpgrade.DirDescription`의 반환값/모델 전체를 번역하는 방식이나 `File.ReadAllText` 전역 패치는 권장하지 않는다. 표시문과 식별자가 같은 경로에 섞여 있다.

## 12. 필요한 게임 / Unity 참조 DLL

### 12.1 패치 프로젝트의 최소 후보

| 참조 | 용도 / 확보 경로 |
|---|---|
| `TeamSoda.Duckov.Core.dll` | Duckov.Modding.ModBehaviour 및 실제 mod loader 확인. 게임 Managed 폴더에서 확보. |
| `UnityEngine.CoreModule.dll` | GameObject, Transform, Font, Texture2D, Unity Object lifecycle. |
| `Unity.TextMeshPro.dll` | TMP_Text, TextMeshProUGUI, TMP_FontAsset, TMP_Settings. |
| `UnityEngine.AssetBundleModule.dll` | 패치 폰트 AssetBundle 로딩. |
| `UnityEngine.TextRenderingModule.dll` | Unity Font 관련 API. |
| `UnityEngine.IMGUIModule.dll` | GUIStyle 및 원본 맵 IMGUI 폰트. |
| `UnityEngine.UI.dll` | Image 등 UI 타입을 직접 다루는 경우. |
| `UnityEngine.ImageConversionModule.dll` | PNG 교체 자산을 런타임에 로딩할 경우. |
| `UnityEngine.TextCoreFontEngineModule.dll` | FontEngine/TMP 생성 API를 직접 사용하는 경우. |
| `0Harmony.dll` | 원본 AssemblyRef는 **2.4.1.0**. 게임/원본과 호환되는 단일 로드 버전 사용. |
| `Newtonsoft.Json.dll` | 외부 번역 JSON 로딩. 원본 AssemblyRef는 **13.0.0.0**. 기존 게임 제공 버전 호환 확인. |
| `netstandard.dll` 및 해당 target framework 참조 | 원본은 netstandard **2.1.0.0**에 참조한다. 게임 Mono와 mod SDK 계약에 맞는 target을 확인. |
| LocalizationManager가 실제 정의된 assembly | 정확한 assembly는 미확인. `SodaCraft.Localizations.LocalizationManager` 타입 위치와 API signature를 게임에서 확인하거나 reflection으로 호출. |

TMP component의 font만 reflection으로 조작하는지, 자산을 직접 읽는지에 따라 참조 범위가 달라진다. internal 원본 클래스를 reflection으로 찾으면 `学园终端.dll`을 배포물이나 필수 컴파일 참조에 넣을 필요가 없다. 원본 모드·게임 DLL을 패치 배포물에 복제하지 않는다.

### 12.2 원본의 나머지 AssemblyRef

원본 DLL에는 추가로 PhysicsModule, UIModule, AudioModule, VideoModule, ParticleSystemModule, InputLegacyModule, UnityWebRequestModule, UnityWebRequestAudioModule, Unity.InputSystem `1.14.0.0`, AstarPathfindingProject, ItemStatsSystem, Eflatun.SceneReference, UniTask, FMODUnity, TeamSoda.Duckov.Utilities, com.rlabrecque.steamworks.net, websocket-sharp `1.0.1.0` 등의 참조가 있다. 이는 원본 전체 기능의 의존성이지 간단한 한국어 패치가 모두 직접 참조해야 하는 목록은 아니다.

분석용 .NET SDK 8.0.419와 ILSpy 9.1.0.7988은 디컴파일 도구다. 이를 근거로 패치 target framework를 .NET 8로 정하면 안 된다. 실제 게임 Managed DLL, Mono/Unity 버전 및 지원 mod 프로젝트를 확인하여 결정한다.

## 13. 권장 프로젝트 폴더 구조

아래는 향후 생성할 구조이며 이번 작업에서는 생성하지 않았다.

```text
repository/
  ANALYSIS.md
  src/KoreanPatch/
    KoreanPatch.csproj
    ModBehaviour.cs
    TranslationCatalog.cs
    OriginalModAdapter.cs
    Patches/                   공통·화면별·맵 표시 hook
    Fonts/                     TMP/IMGUI 수명 관리
    Assets/                    원본 key → 패치 texture 매핑
  package/学园终端_KoreanPatch/
    info.ini                   loader 계약 확인 후 확정
    KoreanPatch.dll            loader 계약 확인 후 이름 확정
    translations/ko-KR.json
    fonts/korean-fonts.bundle   TMP_FontAsset + Unity Font
    fonts/LICENSE.txt
    textures/                  교체가 필요한 이미지에만 사용
  tools/                       사전·placeholder·compatibility 검사
  references/                  로컬 게임 참조; 배포·Git 추적 제외
```

번역 문서 편집과 DLL 구현을 분리한다. 번역 JSON의 schema/ID는 안정적으로 유지하고 실제 문구 변경에는 DLL 빌드가 필요 없게 한다. 원본 경로를 닮은 한국어 디렉터리를 만들거나 원본 소재 폴더에 파일을 덮어쓰지 않는다.

## 14. 실제 구현 순서와 필요한 검증

1. 실제 게임 Managed DLL 및 loader를 확보한다. info.name/assembly/namespace 규칙, 게임 언어 API, Unity/TMP/Mono/Harmony 버전을 확인한다.
2. 이번 DLL의 타입·signature 검증 목록을 바탕으로 compatibility probe와 원본 assembly 발견 lifecycle을 만든다. 지원되지 않는 hook은 원문 fallback으로 처리한다.
3. 외부 사전 schema 및 원문·ID·문맥 조회기를 만든다. 빈 사전으로 시작하고 placeholder/rich text/JSON 검사와 재로드 기능을 먼저 검증한다.
4. 별도 폰트 bundle을 준비하고 TMP 및 IMGUI의 한글·숫자·기호·중국어 혼합 표시, atlas 증가, 원본 clear/reload, 패치 해제 수명을 확인한다.
5. `Ui.Text` 생성 hook과 표시 component registry를 만든다. 게임 기본 UI, 입력, 사용자 이름·채팅은 제외한다. 모델·save·payload가 변하지 않음을 확인한다.
6. QuestManager의 템플릿, 모집 확률, 메일·공지, 캐릭터·재료명 등 문맥별 표시 adapter를 추가한다. PackConfirmModal의 원문 비교 예외를 먼저 다룬다.
7. 동적 .text 갱신을 검수한 뒤 개별 출력 hook 또는 scoped setter 방식을 확정한다. 화면 열고 닫기 및 선택·금액·시간·피드백 업데이트를 검사한다.
8. 실제 Localization key와 맵 IMGUI를 처리한다. 한국어 이외 언어, 언어 변경, 원본 override 재등록도 확인한다.
9. 이미지 전수 검수로 C 목록을 확정하고 필요한 자산만 패치 Texture2D로 교체한다. 동영상·맵 자산은 지원 방법을 별도 확인한다.
10. 위 기반이 안정된 후 실제 번역을 작성한다. JSON 파일만 바꾸고 사전 재로드로 반영되는지 확인한다.
11. 원본 업데이트와 로드 순서·비활성화 조합을 테스트한 뒤 별도 모드로 패키징한다.

게임 내 필수 확인 사례:

- 원본만 실행했을 때와 패치 적용 시 아이템·선물·재료 가방 key, 가격·수량, 임무 진행·보상, 모집 확률·소비가 같은가.
- 구매 상세에서 같은 내용물을 두 번 누를 때 원본처럼 닫히는가.
- 캐릭터 선택·스킬/장비 탭·재료 툴팁·상점 필터·모집 배너·임무 갱신·메일/공지에서 번역 및 레이아웃이 유지되는가.
- 사용자 이름·채팅·입력 문자열과 원격 payload는 그대로인가.
- 패치가 먼저/나중에 로드되는 경우, 원본 재활성화, 한국어 전환, patch 해제 후 폰트·override가 정상인가.
- 번역 누락·잘못된 JSON·미지원 원본 signature에서는 기능 실패 대신 원문을 보여 주는가.

이번 작업에서 위 게임 실행 검증은 하지 않았다. 분석 환경에는 실제 게임 Managed DLL 및 실행 환경이 없으며, 원본 데이터를 실행·변경하지 않는 정적 분석만 수행했다.

## 15. 원본 업데이트 시 깨질 가능성과 대응

| 변경 | 위험 | 대응 |
|---|---|---|
| internal 클래스·메서드 이름/signature 변경 | Harmony 탐색 실패 | 완전한 타입+signature 검증. 실패 hook만 비활성화하고 원문 fallback. token을 고정 ID로 쓰지 않는다. |
| UI 생성 함수 변경 또는 SetText 사용 시작 | 출력 누락 | 신규 호출 경로를 점검하고 component registry·화면별 adapter를 갱신한다. |
| 같은 Name이 새 로직 비교에 사용됨 | 표시 치환이 분기까지 영향 | `.text` 읽기 및 모델 비교 경로를 새 버전마다 확인한다. 논리용 component는 whitelist에서 제외한다. |
| JSON record ID·분류·경로 변경 | 사전 연결 실패 | 원본 ID 기반 조회에 source hash/원문 검증 추가. 새 ID는 원문. 임의 이름 변환으로 연결하지 않는다. |
| 문구/템플릿 변경 | 잘못된 번역·placeholder 손실 | 템플릿 signature·원문 hash 확인. 합성 문자열의 광범위 regex에 의존하지 않는다. |
| 폰트 cache, Clear, bundle 구조 변경 | 누락 글리프 또는 파괴 asset 참조 | 원본 폰트 lifecycle hook 재검증, 자기 소유 자산만 관리, default 실패 경로도 검사. |
| Localization registration·언어 변경 | 원본이 한국어 override 덮어씀 | 원본 registration 이후 재적용 및 key별 이전 상태 복원. |
| 새 원격 메일/공지·새 이미지 | 번역 누락 | 명시적 source/ID 사전과 원문 fallback, 별도 C 목록 갱신. |
| 게임 Unity/TMP/Mono/Harmony 업데이트 | bundle/패치 로딩 실패 | 게임 버전별 참조·bundle·signature 확인. Harmony 중복 배포를 피한다. |

원본 DLL hash가 달라졌다고 무조건 전체 기능을 정지시키기보다는 signature와 필요한 필드 검증으로 기능별 호환 여부를 판단한다. 다만 지원 버전·검증되지 않은 변경은 명확히 기록한다. compiler-generated lambda, 필드 배치, 명령 위치 기반 transpiler는 변경에 약하므로 최소화한다.

## 16. 추가 확인 사항과 분석의 한계

- 실제 게임 Managed DLL, game mod loader 및 언어 선택 API가 필요하다. 게임 참조 없이 패치 빌드·동작을 확정할 수 없다.
- `商店/商店功能.json`, 코드가 기대하는 일부 가방 fallback과 실제 압축 경로의 불일치를 설치된 Workshop 최신본과 대조해야 한다. 이번에 원본 경로를 고치지 않았다.
- reference JSON/CSV가 외부 도구나 다른 DLL의 간접 스캔에 쓰이는지 런타임에서 확인해야 한다. 파일명이 DLL 문자열에 없다는 사실만으로 미사용을 확정하지 않는다.
- 모든 902개 이미지와 GIF/MP4, map bundle의 texture/material에 대한 전수 시각 검수는 별도 작업이다. 직접 확인한 모집 배너 외에 C 항목이 더 있을 수 있다.
- 한국어 폰트 재배포 라이선스 및 게임 버전에 맞는 AssetBundle 제작·런타임 로딩을 검증해야 한다. 원본 assets의 NEXON Games / Yostar 저작권 표기도 패치 이미지 배포 전에 확인한다.
- `PackConfirmModal` 원문 비교, label 폭, 기존 text에 대한 append/비교, 사용자 입력 제외는 실제 실행으로 회귀 검증해야 한다.
- 늦게 로드되는 패치가 이미 생성한 화면을 어떻게 갱신하고 언로드 시 원래 상태로 돌아갈지는 게임 lifecycle에서 확인한다.
- 원격 콘텐츠의 전체 목록·변화, 실제 서버 응답을 분석한 것은 아니다.

## 부록 A. 정적 분석 환경과 재현 자료

setup 스킬에 따라 실제 분석에 필요한 도구를 저장소 밖에 준비했다. Python 환경은 py7zr, dnfile, dncil, UnityPy, fonttools, json5, Pillow를 사용했다. .NET SDK 8.0.419 및 ILSpy 9.1.0.7988로 원본 DLL을 읽었다. 압축 검사·JSON 파싱·폰트 cmap 및 bundle 구조 판독·DLL metadata 조회가 실제 완료된 분석 환경 검증이다. 게임 실행 준비 완료를 뜻하지 않는다.

이번 세션의 자료 위치는 다음과 같다. 이 자료는 저장소 배포물에 추가하지 않았으므로 다른 클라우드 인스턴스에서는 첨부 파일과 도구를 다시 준비해야 한다.

```text
/workspace/scratch/mod-investigation/
  original.7z
  original/3812418766/          변경하지 않은 분석용 원본
  manifest-before.json         1,371개 원본 파일 SHA-256
  archive-list.json
  decompiled/                  ILSpy 출력
  readable/                    읽기용 파생 출력
  json-schema-index.json       JSON 97개 최상위 key 목록
  target-metadata.json         주요 타입/메서드/token/signature
  text-write-methods.json       직접 .text 대입 조사 자료
```

## 부록 B. 전체 JSON 파일 목록

97개 파일의 이름과 구조를 모두 확인했다. 아래 목록은 원본 상대 경로를 그대로 기록한 것이며, 파일명 번역·이동 제안이 아니다. 스킬 효과의 중국어 값·key는 기능 데이터로 보존한다. 사용 여부가 불명확한 자료는 본문 5절의 한계를 적용한다.

- `board.json`
- `buy_pyroxene.json`
- `cafe.json`
- `characters.json`
- `config.json`
- `crafting.json`
- `shop.json`
- `upgrade.json`
- `weapons.json`
- `≤╫Ω╚.json`
- `▀┬∩┴/▀┬∩┴═φ╥÷.json`
- `█╬°╨/╘│╬².json`
- `业务区/主界面/业务区.json`
- `业务区/学院交流会/学院交流会.json`
- `业务区/悬赏通缉/悬赏通缉.json`
- `业务区/特别委托/特别委托.json`
- `任务/quests.json`
- `公告/01 欢迎来到学园终端.json`
- `公告/02 学园终端 · 功能总览.json`
- `公告/03 学园终端开服纪念.json`
- `公告/_模板（复制我）.json`
- `商店/商店分类/10_限时.json`
- `商店/商店分类/20_青辉石.json`
- `商店/商店分类/30_礼包.json`
- `好感礼物.json`
- `学生数据库.json`
- `技能升级.json`
- `技能模板/基础/效果.json`
- `技能模板/学生技能数据库/BA_完整技能数据库_可交给MOD_AI/BA_学生技能_完整清洗版.json`
- `技能模板/必杀/效果.json`
- `技能模板/被动/效果.json`
- `技能模板/辅助/效果.json`
- `抽卡x招募/招募.json`
- `新场景地图/maps.json`
- `新场景地图/stages.json`
- `新场景地图/地图配置.json`
- `背包/装备.json`
- `装备强化.json`
- `角色/一之濑明日奈/技能/基础/效果.json`
- `角色/一之濑明日奈/技能/必杀/效果.json`
- `角色/一之濑明日奈/技能/被动/效果.json`
- `角色/一之濑明日奈/技能/辅助/效果.json`
- `角色/丰见小鸟/技能/基础/效果.json`
- `角色/丰见小鸟/技能/必杀/效果.json`
- `角色/丰见小鸟/技能/被动/效果.json`
- `角色/丰见小鸟/技能/辅助/效果.json`
- `角色/丹花伊吹/技能/基础/效果.json`
- `角色/丹花伊吹/技能/必杀/效果.json`
- `角色/丹花伊吹/技能/被动/效果.json`
- `角色/丹花伊吹/技能/辅助/效果.json`
- `角色/天童爱丽丝/技能/基础/效果.json`
- `角色/天童爱丽丝/技能/必杀/效果.json`
- `角色/天童爱丽丝/技能/被动/效果.json`
- `角色/天童爱丽丝/技能/辅助/效果.json`
- `角色/宇泽玲纱/技能/基础/效果.json`
- `角色/宇泽玲纱/技能/必杀/效果.json`
- `角色/宇泽玲纱/技能/被动/效果.json`
- `角色/宇泽玲纱/技能/辅助/效果.json`
- `角色/小钩晴/技能/基础/效果.json`
- `角色/小钩晴/技能/必杀/效果.json`
- `角色/小钩晴/技能/被动/效果.json`
- `角色/小钩晴/技能/辅助/效果.json`
- `角色/小鸟游星野/技能/基础/效果.json`
- `角色/小鸟游星野/技能/必杀/效果.json`
- `角色/小鸟游星野/技能/被动/效果.json`
- `角色/小鸟游星野/技能/辅助/效果.json`
- `角色/早濑优香/技能/基础/效果.json`
- `角色/早濑优香/技能/必杀/效果.json`
- `角色/早濑优香/技能/被动/效果.json`
- `角色/早濑优香/技能/辅助/效果.json`
- `角色/杏山和纱/技能/基础/效果.json`
- `角色/杏山和纱/技能/必杀/效果.json`
- `角色/杏山和纱/技能/被动/效果.json`
- `角色/杏山和纱/技能/辅助/效果.json`
- `角色/砂狼白子/技能/基础/效果.json`
- `角色/砂狼白子/技能/必杀/效果.json`
- `角色/砂狼白子/技能/被动/效果.json`
- `角色/砂狼白子/技能/辅助/效果.json`
- `角色/空崎日奈/技能/基础/效果.json`
- `角色/空崎日奈/技能/必杀/效果.json`
- `角色/空崎日奈/技能/被动/效果.json`
- `角色/空崎日奈/技能/辅助/效果.json`
- `角色/霞泽美游/技能/基础/效果.json`
- `角色/霞泽美游/技能/必杀/效果.json`
- `角色/霞泽美游/技能/被动/效果.json`
- `角色/霞泽美游/技能/辅助/效果.json`
- `角色/鬼方佳代子/技能/基础/效果.json`
- `角色/鬼方佳代子/技能/必杀/效果.json`
- `角色/鬼方佳代子/技能/被动/效果.json`
- `角色/鬼方佳代子/技能/辅助/效果.json`
- `角色/鹫见芹娜/技能/基础/效果.json`
- `角色/鹫见芹娜/技能/必杀/效果.json`
- `角色/鹫见芹娜/技能/被动/效果.json`
- `角色/鹫见芹娜/技能/辅助/效果.json`
- `邮件/10.3日BUG修复补偿.json`
- `邮件/_模板（复制我）.json`
- `邮件/欢迎来到学园终端.json`
