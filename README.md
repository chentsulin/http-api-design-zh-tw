# HTTP API 設計指南

## 介紹

這份指南描述一系列 HTTP+JSON 的 API 設計經驗，這些最初來自於 [Heroku 平台 API](https://devcenter.heroku.com/articles/platform-api-reference) 的工作經歷。

這份指南補強了原本的 API，也對 Heroku 新的內部
API 起到了引導的作用。我們希望 Heroku 外面的 API 設計者也會對此感興趣。

我們的目標是在保持一致性與關注業務邏輯的同時，也能避免設計歧異。我們在尋找_一個良好、一致並充分文件化的方法_來設計 API，而不需要是_唯一/理想化的方法_。

本文假設你已經熟悉 HTTP+JSON API 的基礎知識，因此不會在這份指南中涵蓋所有的基礎概念。

歡迎對這份指南做出[貢獻](CONTRIBUTING.md)。

## 內容

* [基礎](#基礎)
  *  [關注點分離](#關注點分離)
  *  [要求安全的連接](#要求安全的連接)
  *  [在 Accept 標頭中指定版本](#在-accept-標頭中指定版本)
  *  [支援 ETag 來快取](#支援-etag-來快取)
  *  [提供 Request-Id 用以追蹤檢討](#提供-request-id-用以追蹤檢討)
  *  [用範圍把大型回應拆分成多個請求](#用範圍把大型回應拆分成多個請求)
* [請求](#請求)
  *  [回傳適當的狀態碼](#回傳適當的狀態碼)
  *  [盡可能提供完整的資源](#盡可能提供完整的資源)
  *  [接受用 JSON 編碼的請求本體](#接受用-json-編碼的請求本體)
  *  [使用一致的路徑格式](#使用一致的路徑格式)
  *  [小寫的路徑和屬性](#小寫的路徑和屬性)
  *  [支援非 id 的取用給予方便](#支援非-id-的取用給予方便)
  *  [最小化路徑巢狀](#最小化路徑巢狀)
* [回應](#回應)
  *  [提供 (UU)ID 給資源](#提供-uuid-給資源)
  *  [提供標準的時間戳記](#提供標準的時間戳記)
  *  [使用 ISO8601 中格式化的 UTC 時間](#使用-iso8601-中格式化的-utc-時間)
  *  [巢狀的外鍵關係](#巢狀的外鍵關係)
  *  [產生結構化的錯誤](#產生結構化的錯誤)
  *  [顯示頻率限制狀態](#顯示頻率限制狀態)
  *  [在所有回應中最小化 JSON](#在所有回應中最小化-json)
* [文件](#文件)
  *  [提供機器可讀的 JSON schema](#提供機器可讀的-json-schema)
  *  [提供人可讀的文件](#提供人可讀的文件)
  *  [提供可執行的範例](#提供可執行的範例)
  *  [描述穩定度](#描述穩定度)

### 基礎

#### 關注點分離

保持事情簡單，同時用請求與回應週期的不同部分之間關注點分離的方式來設計。保持簡單的原則讓我們能專注在更大、更難的問題上面。

產生的請求與回應應該指向特定資源或資源集合。使用路徑來表示資源，使用主體來傳輸內容，使用標頭來溝通元資料 (metadata)。在特殊案例中，查詢參數也可以作為一個傳遞標頭資訊的手段，但還是應該優先使用標頭，因為它更靈活並且可以傳達更多樣化的資訊。

#### 要求安全的連接

要求使用 TLS 建立安全的連接來訪問 API，沒有特例。不值得嘗試找出或解釋什麼時候適合使用 TLS，而什麼時候不適合。讓全部都使用 TLS。

理想情况下，為了避免任何不安全的資料交換，應該簡單地拒絕任何 http 或 80 埠的非 TLS 請求，並不進行回應。在不能這樣做的環境下，則回傳 `403 Forbidden`。

不建議使用重導向，因為它允許馬虎和不好的客服端行為，也沒有任何明確的好處。客戶端依賴重導向會使伺服器的流量倍數增長，且會因為已經在第一次呼叫過程中暴露敏感資料而使 TLS 沒有用。

#### 在 Accept 標頭中指定版本

版本化和版本間的轉換是設計與營運 API 中最具有挑戰性的方面之一。因此，最好一開始就使用一些適當的機制來減少麻煩。

為了避免重大改變會讓使用者的程式發生意外，所有請求最好都要指定版本。應該避免使用預設的版本，因為它在未來會非常難以變更。

最好的方式是在標頭中提供版本規格還有其他的元資料 (metadata)，使用 `Accept` 標頭跟客製化的內容類型，例如：

```
Accept: application/vnd.heroku+json; version=3
```

#### 支援 ETag 來快取

在所有回應中加上 `ETag` 來識別回傳資源的特定版本。這讓使用者可以快取資源，並把這個值放在 `If-None-Match` 標頭中來做出請求，依此決定是否應該更新快取。

#### 提供 Request-Id 用以追蹤檢討

在每個 API 回應中加上 `Request-Id` 標頭，附上一個 UUID 的值。藉由在客戶端、伺服器和任何依賴的服務中記錄這些值，可以提供一個機制來對請求追蹤、診斷和除錯。

#### 用範圍把大型回應拆分成多個請求

大型回應應該拆分成多個請求，使用 `Range` 標頭來指定何時有更多資料可以使用，以及要如何去取得。參閱 [Heroku 平台 API 對 Range 的討論](https://devcenter.heroku.com/articles/platform-api-reference#ranges) 來了解請求與回應標頭、狀態碼、限制次數、排序和迭代的細節。

### 請求

#### 回傳適當的狀態碼

隨著每個回應回傳適當的 HTTP 狀態碼。成功的回應應該依照這份指南給予狀態碼：

* `200`：`GET` 請求成功、`DELETE` 或 `PATCH` 請求同步地完成，或是 `PUT` 請求同步地更新了存在的資源時
* `201`：`POST` 請求同步地成功完成，或是 `PUT` 請求同步地建立了新的資源時
* `202`：已經接受將會被非同步處理的 `POST`、`PUT`、`DELETE` 或 `PATCH` 請求
* `206`：`GET` 請求已經成功，但只有回應一部分：參閱[前面提到範圍的部分](#divide-large-responses-across-requests-with-ranges)

使用認證和授權錯誤碼必須小心：

* `401 Unauthorized`：因為使用者沒有認證，所以請求失敗
* `403 Forbidden`：因為使用者沒有權限可以訪問特定的資源，所以請求失敗

當發生錯誤的時候，回傳適合的狀態碼以提供額外的資訊：

* `422 Unprocessable Entity`：你的請求已經被解析，但其中包含不合法的參數
* `429 Too Many Requests`：你已經到了使用頻率上限，稍後再試
* `500 Internal Server Error`：伺服器發生了一些錯誤，檢查網站狀態並回報問題

參考 [HTTP 回應碼規格](https://tools.ietf.org/html/rfc7231#section-6) 以了解使用者端錯誤和伺服器錯誤情況下的狀態碼。

#### 盡可能提供完整的資源

在可能的情況下，在回應中提供完整的資源表示 (換句話說，包含所有屬性的物件)。總是在 200 和 201 回應中提供完整的資源，包括 `PUT`/`PATCH` 和 `DELETE`
請求，例如：

```bash
$ curl -X DELETE \
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 回應將不會包含完整的資源表示，例如：

```bash
$ curl -X DELETE \
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 接受用 JSON 編碼的請求本體

在 `PUT`/`PATCH`/`POST` 請求接受用 JSON 編碼的請求本體，作為表單編碼資料的替代或補充。這建立了與 JSON 編碼的回應本體之間的對稱，例如：

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 使用一致的路徑格式

##### 資源名稱

使用複數版本的資源名稱，除非資源是單體存在於系統之中 (例如，在大多數系統中一個給定的使用者將只會擁有一個帳戶)。這讓參考特定的資源的方式可以保持一致。

##### 操作

對於不需要任何特別的操作的個別資源，應該用端點 (endpoint) 佈局。在需要特別操作的情快下，把他們放置在標準的 `actions` 前綴後面，來清楚的界定他們：

```
/resources/:resource/actions/:action
```

例如

```
/runs/{run_id}/actions/stop
```

#### 小寫的路徑和屬性

使用小寫、中線分隔的路徑名稱，與主機名稱一致，例如：

```
service-api.com/users
service-api.com/app-setups
```

屬性也使用小寫，但使用底線來分隔，這樣的話在 JavaScript 中可以不用括號打出屬性名稱，例如：

```
service_class: "first"
```

#### 支援非 id 的取用給予方便

在某些情況下，讓終端使用者提供 ID 來辨識資源或許不是很方便。例如，使用者想的或許是 Heroku 的應用程式名稱，但是那個應用程式可能是用 UUID 來辨識的。在這種情況下，你可能會想要同時接受 id 或名稱，例如：

```bash
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

不要只接受名稱而排除 ID。

#### 最小化路徑巢狀

在資料模型中有父子的巢狀資源關係，路徑可能會變成深度巢狀，例如：

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

應該藉由在根路徑定位資源，來限制巢狀的深度。使用巢狀來表示一個範疇中的集合。例如，上面的例子中，dyno 屬於一個 app，app 屬於一個 org：

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### 回應

#### 提供 (UU)ID 給資源

預設給每個資源一個 `id` 屬性。使用 UUID，除你你有非常好的理由不這樣做。不要使用不是跨越機器或服務全域唯一的 ID，特別是自動遞增的 ID。

用小寫的 `8-4-4-4-12` 格式來輸出 UUID，例如：

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 提供標準的時間戳記

預設提供 `created_at` 與 `updated_at` 時間戳記給資源，例如：

```javascript
{
  // ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  // ...
}
```

這些時間戳記對某些資源或許沒有意義，在那樣的情況下可以省略。

#### 使用 ISO8601 中格式化的 UTC 時間

只用 UTC 接受和回傳時間。用 ISO8601 格式輸出時間，例如：

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### 巢狀的外鍵關係

序列化外鍵參考為一個巢狀物件，例如：

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

而不是：

```javascript
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

用這個方法來加入更多關聯資源的資訊，使不必改變回應的結構或加入更多頂層的回應欄位變得可能，例如：

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  // ...
}
```

#### 產生結構化的錯誤

再發生錯誤時，產生一致、結構化的回應本體。包含一個機器可讀的錯誤 `id`、一個人可讀的錯誤 `message` 和選擇性地提供 `url` 指引客戶端找到更多有關錯誤和如何解決它的資訊，例如：

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

文件化你的錯誤格式和客戶端可能會遇到的錯誤 `id`。

#### 顯示頻率限制狀態

限制客戶端的請求頻率以保護服務的健壯，並維持其他客戶端的高服務品質。你可以使用 [token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) 來量化請求限制。

在每個請求的 `RateLimit-Remaining` 回應標頭中回傳請求 token 的剩餘次數。

#### 在所有回應中最小化 JSON

多餘的空白會增加回應請求的大小，而且很多給人看的客戶端都會自動地「美化」JSON 輸出。所以最好在回應中最小化 JSON，例如：

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z","created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

而不是：

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

你可以考慮選擇性地提供一個方法讓客戶端取回更詳細的回應，可以藉由查詢參數 (例如：`?pretty=true`) 或者是藉由 `Accept` 標頭參數 (例如：`Accept: application/vnd.heroku+json; version=3; indent=4;`)。

### 文件

#### 提供機器可讀的 JSON schema

提供機器可讀的 schema 來明確地指定 API。使用 [prmd](https://github.com/interagent/prmd) 來管理你的 schema，並確保
它使用 `prmd verify` 驗證過。

#### 提供人可讀的文件

提供人可讀的文件讓客戶端開發者可以用它來了解你的 API。

如果你如上所述的使用 prmd 建立 schema，你可以輕鬆地用 `prmd doc` 產生所有端點的 Markdown 文件。

附帶的端點細節，應該提供 API 的包含以下資訊的概覽：

* 認證，包括獲得和使用認證 token。
* API 穩定度和版本狀況，包含要如何選擇想要的 API 版本。
* 常用的請求和回應標頭。
* 错误的格式。
* 不同客戶端程式語言的 API 使用範例。

#### 提供可執行的範例

提供可執行的範例讓使用者可以直接地在終端機輸入它來看可執行的 API 呼叫。為了最大的可擴展性，這些例子應該可以逐字使用，以最小化使用者嘗試 API 所需要做的事，例如：

```bash
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

如果你使用 [prmd](https://github.com/interagent/prmd) 來產生 Markdown 文件，你將會很容易地獲得每個端點的範例。

#### 描述穩定度

描述你的 API 的穩定度和它多個端點的成熟度和穩定度，例如：伴隨著原型/開發/產品標誌。

參閱 [Heroku API 相容性政策](https://devcenter.heroku.com/articles/api-compatibility-policy) 以了解可能的穩定度和變更管理的方法。

一旦 API 宣告為生產環境適用且為穩定的，在此 API 版本中不要做不向下相容的變更。如果你需要不向下相容的變更，用遞增的版本號建立一個新的 API。

