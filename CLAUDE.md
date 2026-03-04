# Diamond Manager (판매 관리 대장) — 프로젝트 메모리

## 기본 정보
- **파일**: `D:\파이썬\diamond_manager.py` (약 5,600줄, 단일 파일 Tkinter 앱)
- **현재 버전**: v1.0.65
- **언어**: Python 3.14 + Tkinter (단일 .py 파일 구조)
- **용도**: 다이아몬드/게임 아이템 판매 관리 대장 (거래 입력, 수익 계산, 통계, 완료 내역 관리)
- **설치 경로**: `D:\Program Files (x86)\DiamondManager\`
- **GitHub**: `https://github.com/kmseuk/sales-manager` (Public, Releases로 업데이트 배포)

## 빌드 파이프라인
1. `change_summary.txt` 작성 (변경 내용 한 줄 요약)
2. `py _trigger_log.py` — 버전 bump + update_log.txt 갱신 + 백업 + Output/version.txt 복사
3. `py -m PyInstaller --noconfirm --clean DiamondManager.spec` — EXE 빌드
4. `"C:\Users\kimse\AppData\Local\Programs\Inno Setup 6\ISCC.exe" "D:\파이썬\DiamondManager_setup.iss"` — 설치파일 생성
5. **결과물**: `D:\파이썬\Output\DiamondManager_Setup.exe`
6. (선택) GitHub Releases에 새 버전 업로드: `version.txt` + `DiamondManager_Setup.exe`

> **주의**: 이 시스템에서는 `python`이 아닌 `py` 명령어 사용
> **주의**: Inno Setup 빌드 시 백신이 Output 폴더를 잠그면 실패할 수 있음 → 5초 후 재시도

## 프로젝트 파일 구조
```
D:\파이썬\
├── diamond_manager.py          # 메인 앱 소스 (5,600줄)
├── version.txt                 # 현재 버전 (1.0.63)
├── trade_data.csv              # 판매중 거래 데이터
├── completed.csv               # 완료된 거래 데이터
├── settings.json               # 사용자 설정 (테마, 세율, 색상 등)
├── column_config.json          # 컬럼 너비 설정
├── icon.ico                    # 앱 아이콘
├── DiamondManager.spec         # PyInstaller 스펙 파일
├── DiamondManager_setup.iss    # Inno Setup 스크립트
├── _trigger_log.py             # 빌드 헬퍼 (버전 bump + 로그 + 백업 + Output 복사)
├── change_summary.txt          # 빌드 시 변경 내용 요약
├── update_log.txt              # 버전별 변경 로그
├── 서버_업데이트_전환_코드.py   # 서버 기반 업데이트 전환 참조 코드 (로컬→서버 전환 가이드)
├── logs/                       # 로그 폴더 (app/error/change, 30일 유지)
├── backup/                     # 버전별 자동 백업
├── dist/DiamondManager/        # PyInstaller 빌드 출력
└── Output/                     # 배포 폴더 (Setup.exe + version.txt)
```

## 코드 구조 (diamond_manager.py)

### 전역 영역 (1~950줄)
| 줄 범위 | 내용 |
|---------|------|
| 1~8 | import (tkinter, pandas, sys, os, json, threading, tempfile, subprocess, urllib 등) |
| 13~260 | **버전/백업 시스템**: `auto_backup()`, `auto_version()`, `_bump_version()`, `is_file_changed()`, `generate_update_log()` — 개발 시 자동 버전 관리 |
| 262 | `APP_VERSION = auto_version()` |
| 267~268 | `SETTINGS_FILE`, `UPDATE_URL` (GitHub Releases URL) |
| 278~308 | **로그 시스템**: `_make_logger()` → `log_app`, `log_err`, `log_chg` (일별 분리, 30일 유지) |
| 312~366 | **색상 팔레트 C dict** + **테마 프리셋 THEMES** (Ocean Deep, Midnight Purple, Forest Green, Warm Dark) |
| 373~400 | **DEFAULT_SETTINGS** — 세율, 테마, 행 색상, 단가, 도킹, 스크롤 위치, 업데이트 설정 등 |
| 404~414 | `_resource_path()`, `ICON_PATH`, `FONT = "Malgun Gothic"` |
| 417~462 | `_apply_icon()`, `_get_win_build()`, `_apply_dark_titlebar()` (Win10/11 다크 타이틀바) |
| 464~533 | **`_custom_msgbox()`** — 커스텀 메시지박스 (info/error/yesno, 다크 테마 적용) |
| 535~640 | 유틸리티: `safe_eval()`, `safe_float()`, `to_num()`, `fmt_num()`, `safe_int()`, `clean_product_name()`, `build_sort_key()` |
| 644~815 | **`AutocompleteEntry`** 클래스 — 자동완성 입력 위젯 (상품명 검색) |
| 822~950 | **`DockableWindow`** 클래스 — 메인 창 좌/우 도킹 서브 창 (스냅, 드래그 복원) |

### DiamondApp 클래스 (955~5553줄)

#### 데이터 컬럼 정의
- `ALL_COLS`: ID, 날짜, 상품명, 구매수량, 구매총액, 개당구매, 획득수량, 개당판매, 총판매금액, 정산금, 수익, 상태, _uuid
- `COLS`: UI에 표시되는 컬럼 (ID/날짜/_uuid 제외)
- `NUM_COLS`: 숫자 컬럼 8개 (구매수량, 구매총액, 개당구매, 획득수량, 개당판매, 총판매금액, 정산금, 수익)
- `USER_INPUT`: 사용자 직접 입력 (구매수량, 구매총액, 획득수량, 개당판매)
- `AUTO_CALC`: 자동 계산 (개당구매, 총판매금액, 정산금, 수익)
- `EDIT_COLS`: 인라인 편집 가능 컬럼

#### 핵심 메서드 맵
| 메서드 | 기능 |
|--------|------|
| **`__init__`** | 앱 초기화 (창 설정, 데이터 로드, UI 빌드, 업데이트 체크) |
| **`load_data`** | CSV → DataFrame 로드 (_uuid 자동 부여, 숫자 콤마 제거) |
| **`_load_completed`** | completed.csv 로드 (숫자 콤마 제거) |
| **`_load_settings`** | settings.json 로드 (없으면 DEFAULT_SETTINGS) |
| **`_save_settings`** | settings.json 저장 |
| **`_apply_theme_to_palette`** | 테마 → C dict 반영 |
| **`_apply_theme_live`** | 실시간 테마 전환 (모든 위젯 색상 업데이트) |
| **`_setup_styles`** | ttk 스타일 설정 (Treeview, Progressbar 등) |
| **`_build_ui`** | **메인 UI 구성** — 상단 통계바, 입력 폼, 버튼, Treeview 테이블, 검색 |
| **`_inline_edit`** | 더블클릭 인라인 편집 시작 |
| **`_inline_edit_at`** | 셀 위치에 Entry 오버레이 |
| **`_right_click_load`** | 우클릭 컨텍스트 메뉴 (편집, 복사, 삭제, 상태 변경, 병합, 분리 등) |
| **`_on_sel`** | 행 선택 시 보정값 계산 |
| **`_auto_calc`** | 입력값 → 자동 계산 (개당구매, 총판매금액, 정산금, 수익) |
| **`_build_stats`** | 상단 통계 바 (미판매 수, 판매중 수/총수, 판매액) — `to_num()` 사용 |
| **`_update_ticker`** | 상단 티커 (판매중/완료/전체 수익 표시) — `to_num()` 사용 |
| **`_sort`** | 컬럼 헤더 클릭 정렬 |
| **`_vba_sort`** | VBA 스타일 다단계 정렬 (상태 → 상품명 → 수량) |
| **`_on_close`** | 앱 종료 (설정 저장, 컬럼 너비 저장, 로그) |
| **`_save_data`** | DataFrame → CSV 저장 |
| **`refresh_table`** | Treeview 새로고침 (정렬, 색상, 하이라이트 복원) |
| **`add_record`** | 새 거래 추가 |
| **`_toggle_edit`** | 수정 모드 토글 |
| **`_toggle_highlighter`** | 형광펜 모드 토글 |
| **`load_for_edit`** | 선택 행 → 입력 폼에 로드 |
| **`copy_record`** | 행 복사 |
| **`delete_record`** | 행 삭제 |
| **`mark_complete`** | 판매완료 처리 → completed.csv 이동 |
| **`_mark_selling`** | 미판매 → 판매중 상태 변경 |
| **`_merge_rows`** | 다중 행 병합 (수량/금액 합산) |
| **`_split_row`** | 행 분리 (수량 분할) |
| **`_open_stock_update`** | 수동 발급 팝업 |
| **`_open_stock_control`** | 재고 관리 팝업 |

#### 업데이트 시스템 (GitHub Releases 기반)
| 메서드 | 기능 |
|--------|------|
| **`_ver_tuple`** | 버전 문자열 → 튜플 비교용 |
| **`_check_for_update`** | GitHub에서 version.txt 확인 + GitHub API로 릴리스 노트(changelog) 가져오기 |
| **`_show_update_dialog`** | 새 버전 알림 (변경사항 표시 + 건너뛰기 체크박스 + 수동 업데이트 안내 문구, parent 지원) |
| **`_run_update`** | HTTP 스트리밍 다운로드 + 진행률 바 + Setup.exe 실행 후 앱 종료 |

> **업데이트 URL**: `https://github.com/kmseuk/sales-manager/releases/latest/download`
> **참조 코드**: `D:\파이썬\서버_업데이트_전환_코드.py` (로컬→서버 전환 가이드)

#### 설정 다이얼로그
- **탭 구성**: 세금 | 테마 | 색상 | 관리
- 세율 설정, 테마 실시간 전환, 행/입력창/선택행 색상 커스텀
- 관리 탭: 업데이트 자동확인 체크박스 + 수동 "업데이트 확인" 버튼 (parent 전달로 포커스 유지)

#### 완료내역 창
- `_open_done()`: 별도 Toplevel + DockableWindow (좌/우 도킹)
- completed.csv 데이터 표시, 검색, 되돌리기 기능

#### 통계표 창
- `_open_stats()`: 상품별 수량/금액 집계, 합계행 — `to_num()` 사용
- DockableWindow 도킹 지원

#### 계산기
- `_open_calc()`: 최대 2개 계산기 창
- 기본 사칙연산 + 키보드 바인딩
- 내부적으로 순수 숫자 문자열 사용 (콤마 문제 없음)

### 진입점
- **중복 실행 방지**: Windows Mutex (`DiamondManager_SingleInstance`)
- PyInstaller EXE 호환: 작업 디렉토리 고정
- `root = tk.Tk()` → `DiamondApp(root)` → `root.mainloop()`

## 주요 기능 요약
1. **거래 CRUD**: 저장/수정/복사/삭제, 인라인 더블클릭 편집
2. **자동 계산**: 개당구매 = 총액/수량, 총판매 = 획득×단가, 정산금 = 총판매-세금, 수익 = 정산-구매총액
3. **상태 관리**: 미판매 → 판매중 → 판매완료 (완료 시 completed.csv 이동)
4. **병합/분리**: 같은 상품 행 병합(수량 합산), 수량 분할(행 분리)
5. **정렬**: 컬럼 헤더 클릭 정렬 + VBA 다단계 정렬
6. **형광펜**: 클릭으로 행 강조, Backspace 되돌리기
7. **검색**: 상품명 자동완성 + 실시간 필터
8. **테마**: 4종 프리셋 + 커스텀 색상 (실시간 전환)
9. **도킹**: 완료내역/통계표 창 좌/우 스냅 도킹
10. **창 고정**: 메인/계산기 항상 위 토글
11. **자동 업데이트**: GitHub Releases 기반, 시작 시 버전 확인, 변경사항(릴리스 노트) 표시, 건너뛰기, 진행률 다운로드, Setup.exe 실행
12. **통계**: 상단 티커(수익 합계), 상품별 통계표
13. **로그**: 일별 분리 (app/error/change), 30일 자동 보관
14. **중복 실행 방지**: Windows Mutex
15. **자동 백업**: 개발 시 버전별 .py 파일 백업

## 숫자 처리 규칙
- **`fmt_num()`**: 숫자 → 콤마 포맷 문자열 (예: `6303` → `"6,303"`). DataFrame 저장 및 UI 표시에 사용
- **`to_num()`**: Series 콤마 제거 후 숫자 변환. 모든 합계/통계 계산에 사용 (콤마 안전)
- **`safe_float()`**: 단일 값 콤마 제거 후 float 변환
- **로드 시**: `load_data()`, `_load_completed()`에서 NUM_COLS 콤마 자동 제거

## 데이터 흐름
```
사용자 입력 → add_record() → self.df (DataFrame) → _save_data() → trade_data.csv
                                                  → refresh_table() → Treeview UI
판매완료 → mark_complete() → completed_df → _save_completed() → completed.csv
업데이트 → _check_for_update() → GitHub version.txt + API(릴리스 노트) → _show_update_dialog(changelog) → _run_update() → Setup.exe
```

## 설정 저장 구조 (settings.json)
```json
{
  "tax_rate": 8.0, "theme": "Midnight Purple",
  "row_even": "#252545", "row_odd": "#2a2a50",
  "data_bg": "#1a2332", "data_fg": "#FFFFFF",
  "inp_card_bg": "", "inp_entry_bg": "#e3cff8",
  "sel_bg": "#1a3a5c", "sel_fg": "#ffffff",
  "row_highlight": "#fce4ec", "sep_color": "#3a4a5e",
  "dock_sides": {"done": "left", "stats": "right"},
  "search_bg": "#334155", "search_fg": "#f1f5f9",
  "last_selected_uuid": null, "unit_price": "39",
  "main_topmost": false, "calc_topmost": false,
  "last_sort": null,
  "ui_state.done_scroll": 0.0, "ui_state.done_focus": null,
  "ui_state.stats_scroll": 0.0, "ui_state.stats_focus": null,
  "check_update": true, "skip_update_version": ""
}
```

## 빌드 환경
- **Python**: 3.14.3
- **PyInstaller**: 6.18.0 (`DiamondManager.spec`)
- **Inno Setup**: 6.7.1 (`DiamondManager_setup.iss`)
- **ISCC 경로**: `C:\Users\kimse\AppData\Local\Programs\Inno Setup 6\ISCC.exe`
- **설치 경로**: `D:\Program Files (x86)\DiamondManager\`
- **명령어**: `py` (not `python`)

## 최근 주요 변경 이력
- **v1.0.56**: 업데이트 다이얼로그 "나중에" 버튼 클릭 시 설정창 포커스 복원
- **v1.0.57**: 로컬 폴더 → GitHub Releases 서버 업데이트 전환
- **v1.0.58**: GitHub 요청에 User-Agent 헤더 추가 (403 차단 방지)
- **v1.0.59**: 업데이트 다이얼로그에 수동 업데이트 안내 문구 추가
- **v1.0.60**: 데이터 로드 시 숫자 컬럼 콤마 자동 제거 (판매액 합계 누락 버그 수정)
- **v1.0.61**: `to_num()` 헬퍼 추가, 모든 숫자 계산에 콤마 안전 변환 적용
- **v1.0.62**: 수정 버튼 재클릭 시 커서가 숫자 끝에 위치하도록 수정 (`load_for_edit`에 `icursor(tk.END)` 명시 호출)
- **v1.0.63**: 업데이트 다이얼로그에 GitHub Releases 변경사항(릴리스 노트) 표시 기능 추가 (GitHub API `/releases/latest` → `body` 필드)
- **v1.0.64**: 데이터 입력란 숫자 콤마 포맷 표시 (`_fmt_on_focusout` + `_fmt_guard` 재진입 방지, `load_for_edit` 콤마 포맷 적용)
- **v1.0.65**: 항목값 ▲▼ 버튼 꾹 누르기 자동 반복 (300ms 대기 후 80ms 간격, Leave 시 중지)
