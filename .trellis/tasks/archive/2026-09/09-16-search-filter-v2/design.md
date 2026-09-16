# 搜索过滤补全 — 设计

## 现状

`SearchFilters` 已有 target/sort/duration/start/end。`_PixivSearchRepository._spec` 按 `sort==popularDesc && !isPremium` 路由 popular-preview。`NextPageParser` 有端点参数白名单。sheet 是单文件 `_SearchFilterSheet`(target/sort/duration/日期),search_page 与 search_result_page 共用。

## 新增过滤器面(全部进 `SearchFilters`,wire 契约化)

| 字段 | 类型 | wire | 范围 |
|---|---|---|---|
| sort 扩展 | `popularMaleDesc`/`popularFemaleDesc` | `sort` | illust only;novel 归一 popular_desc(Shaft novelSafe) |
| aiFilter | `SearchAiFilter{all,exclude,only}` | `search_ai_type`(exclude→1) | all/only 省略;only 无服务端参数 → 客户端谓词 |
| bookmarkMin/Max | `int?` | `bookmark_num_min/max` | 会员服务端生效;非会员服务端静默忽略(Shaft 注释证实)→ 客户端谓词兜底 |
| ratio | `SearchRatioPattern{landscape,portrait,square}?` | `ratio_pattern` | illust only |
| contentType | `SearchContentType` | `content_type`(默认 illust_and_manga_and_ugoira 省略) | illust only |
| widthMin/Max, heightMin/Max | `int?` | `width_min/max`,`height_min/max` | illust only |

### 双通道过滤决策

- **服务端**:所有字段进 `toQuery`,非会员也照发(bookmark_num_* 静默忽略已证实;ratio/content_type/width/height 非会员参数)。
- **客户端兜底**:`_SearchFeedController` 覆写 `filterPageIds`,在 super 之后对 IllustSearchQuery 追加谓词——bookmark 区间、`aiOnly`(illust_ai_type==2)。会员/非会员路径统一,幂等(服务端已滤的同集不重滤出错误)。
- novel 侧:`search_ai_type` 的 all/exclude 进 wire;`only` 在 sheet 中仅 illust 类型可见(不发明 novel 客户端谓词)。

### 会员路由扩展

- `SearchSort.isPopular`(popular_desc/male/female)非会员 → popular-preview(illust+novel 同规则)。
- novel 收到 male/female sort 时 `toQuery` 直接归一 `popular_desc`,绝不发非法值。

### 序列化范围

`SearchFilters.toQuery({required word, bool includeIllustParams = false})`:illust-only 字段只在 illust 路径出现。`IllustSearchQuery.toQuery` 传 true;novel 传 false 且走 sort 归一。`toPreviewQuery` 同理。

### cacheKey / cursor

- `cacheKey` 追加全部新字段(位置稳定)。
- `_kNextPageEndpoints` 三处 search 端点白名单补:`search_ai_type`,`bookmark_num_min/max`,`ratio_pattern`,`content_type`,`width_min/max`,`height_min/max`。requiredQuery 全量 pin 不变。

### Sheet UI

`showSearchFilterSheet` 增加 `required SearchResultType type`:
- 排序 chips +2(男向/女向),非会员沿用 popularDesc 的 hint 文案模式。
- 「AI 作品」三 chips:全部/排除 AI/仅 AI(仅 illust type 渲染该组)。
- 「收藏数」两个数字输入(min/max,可单边)。
- 「纵横比」4 chips(不限+3);「作品类别」5 chips(默认+4);「分辨率」4 数字输入。全部仅 illust type。
- 互斥约束不变:duration↔日期照旧。

### 不做

- novel 专属过滤(genre/text_length/word_count/reading_time/is_original)——PRD 未列,记 backlog。
- `users入り` 关键字后缀——客户端谓词已覆盖非会员 bookmark 过滤,不引入第二条桶语义。
- `merge_plain_keyword_results`/`include_translated_tag_results`/`tool`/`lang`——Shaft 有,非 PRD 项。

## 测试

- toQuery:illust 全参发出、novel 省略 illust-only、novel sort 归一、ai only 不出现在 wire、bookmark 单边。
- firstPage 白名单:新参数接受、未知仍拒。
- feed 谓词:bookmark 区间与 aiOnly 过滤进 refill 测试基建。
- repository spec:male/female 非会员走 preview、会员直发。
- sheet:按 type 渲染分组、输入→SearchFilters 断言。
- l10n 四语言。
