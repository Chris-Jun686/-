# 第6章：《变量与语法变化》

## 6.1 关于变量与语法变化

本章讲的是动态文本在跨语言时的处理法则：把句子中会变的内容（如数字、名字、装备名）抽离为带名字的占位符，交由程序填入；而由数量、性别、字符尾音引起的语法变化（如单复数、代词、助词），以及数字、日期、单位的本地化格式，全部交由文本表和格式化引擎处理。目的是让代码只负责传数据，译者能在不破坏程序逻辑的前提下，写出符合目标语言习惯的句子。

## 6.2 规则

### 6.2.1 命名占位符
- **R6-01** 所有动态内容必须使用有明确语义的命名占位符（如 `{player_name}`、`{item_count}`），严禁使用无意义的数字编号（如 `{0}`、`{1}`）。
- **R6-02** 代码必须按占位符名称传参，不得假设占位符在句子中的先后顺序，让译者可以根据目标语言的语法自由调整位置。
- **R6-03** 同一个变量在不同语言的同一句文本中，必须保持占位符名称绝对一致。

### 6.2.2 单复数与量词
- **R6-04** 严禁在代码中使用 `if (count > 1)` 等逻辑判断单复数并拼接后缀（如加 `s`）。
- **R6-05** 必须在文本表内使用语法选择器（如 ICU `plural` 或引擎对应的 Smart Strings 功能）来处理单复数，代码只传入代表数量的变量。
- **R6-06** 对于无需处理复数词尾的语言（如中、日、韩），应直接在模板中嵌入正确的量词（如“把”、“个”、“개”），不依赖代码拼接。

### 6.2.3 性别
- **R6-07** 严禁在代码里根据性别使用 `if-else` 拼接代词或形容词词尾。
- **R6-08** 必须使用语法选择器（如 ICU `select`）在文本表内处理性别分支，代码只传入标准化的性别枚举值（如 `male`、`female`、`other`）。

### 6.2.4 韩语助词
- **R6-09** 当句子需要插入玩家自定义名称（或未知结尾的变量）并带有韩语助词时，程序必须在运行时判断前一个字符的 Unicode 收音，动态计算出正确的助词变量（如 `이/가`、`을/를`），并与名字一起传入文本表。

### 6.2.5 数字、日期与单位格式
- **R6-10** 严禁手动截取数字、手动添加千分位逗号（`,`）或小数点，必须将原始数值交由本地化引擎自动格式化。
- **R6-11** 严禁写死日期格式（如 `YYYY-MM-DD`），必须使用引擎的日期格式化 API，由系统根据目标语言习惯输出。
- **R6-12** 单位（如“里”、“斤”、“度”）必须与数值一起作为格式化单元处理（如 ICU `::measure-unit`），严禁将单位翻译与数值手动拼接连结。

## 6.3 前作发生了什么

### 6.3.1 占位符与语序问题
**问题编号-#15** （位置编号导致异语言语序错乱）
**问题编号-#19** （代码假设了参数传递顺序，修改源文后导致译文张冠李戴）

### 6.3.2 单复数与性别问题
**问题编号-#21** （代码硬编码加 `s`，导致不规则复数直接出错）
**问题编号-#27** （代码用 `if-else` 拼接男女代词，导致译文无法统合）

### 6.3.3 韩语与格式问题
**问题编号-#31** （韩语玩家名加错助词，产生极强机翻感）
**问题编号-#34** （代码写死日期格式，导致海外玩家看到非本地习惯的日期）

## 6.4 正确与错误示范

### 6.4.1 命名占位符与语序调整

**示例：获取武林秘籍**

- 中文原文：你获得了《{book_name}》和 {count} 两银子
- 错误写法：代码中按数字索引传参，并假设语序。
```csharp
// 假设代码中强行规定“书”在前，“银子”在后
msg = Localization.Get("MSG_GET_LOOT", bookName, count);
// 文本表中写：你获得了《{0}》和 {1} 两银子
```
- 正确写法：

文本表（中文）：
| 字符串 ID | 源文 |
|---|---|
| `MSG_GET_LOOT` | 你获得了《{book_name}》和 {count} 两银子 |

文本表（英文）：
| 字符串 ID | 译文 |
|---|---|
| `MSG_GET_LOOT` | You received {count} taels of silver and the book "{book_name}". |

代码：
```csharp
Localization.Get("MSG_GET_LOOT", new { book_name = currentBook, count = silverAmount });
```
> 英文译者将 `{count}` 提到了前面，中文译者把 `{book_name}` 放在前面，程序按名字匹配，互不干扰。

### 6.4.2 单复数与量词

**示例：暗器数量提示**

- 错误写法：在代码里判断单复数并拼接。
```csharp
// 极其错误的写法：强行拼后缀
msg = "You found " + count + " throwing dart" + (count > 1 ? "s." : ".");
```
- 正确写法：

文本表（英文，使用 ICU Plural）：
```icu
MSG_FOUND_DART = You found {count, plural, one {# throwing dart} other {# throwing darts}}.
```

文本表（中文，直接使用量词）：
```icu
MSG_FOUND_DART = 你找到了 {count} 枚飞镖。
```
代码仅传入：
```csharp
Localization.Get("MSG_FOUND_DART", new { count = 5 });
```

### 6.4.3 性别

**示例：NPC 对玩家的尊称**

- 错误写法：代码根据性别用 `if` 拼凑代词。
```csharp
string pronoun = (playerGender == Gender.Male) ? "他" : "她";
msg = pronoun + "的内力深不可测。";
```
- 正确写法：

文本表（中文）：
```icu
MSG_DEEP_QI = {gender, select, male {他} female {她} other {此人}}的内力深不可测。
```

文本表（英文）：
```icu
MSG_DEEP_QI = {gender, select, male {He has} female {She has} other {They have}} unfathomable inner strength.
```
代码传入：
```csharp
Localization.Get("MSG_DEEP_QI", new { gender = playerGender });
```

### 6.4.4 韩语助词

**示例：韩语环境下的拜师提示**

- 错误写法：把助词写死在代码或文本中，导致玩家名字（如“令狐冲”无收音，“岳不群”有收音）读音不同时出现语法错误。
```csharp
// 写死 "이" (有收音)，若玩家叫"令狐冲"，会显示为 "令狐冲이"，极其违和。
msg = "{player_name}이 스승을 만났습니다."; 
```
- 正确写法：

程序层（伪代码）：
```csharp
// 根据玩家名的最后一个字符的 Unicode 编码，计算是带收音还是不带收音
string josa = GetKoreanParticle(playerName, "이", "가");
Localization.Get("MSG_MEET_MASTER", new { player_name = playerName, josa = josa });
```
文本表（韩语）：
```icu
MSG_MEET_MASTER = {player_name}{josa} 스승을 만났습니다.
```
> 若名字是“令狐冲”，传入 `josa="가"`；若名字是“岳不群”，传入 `josa="이"`。

### 6.4.5 数字、日期与单位格式

**示例：轻功飞行距离与时间记录**

- 错误写法：手动截取数字并拼接固定单位。
```csharp
// 错误：写死格式，欧元区玩家会看懵
msg = "今日飞行距离：" + distance.ToString("0.0") + " 里，耗时 " + hours + "时" + minutes + "分。";
```
- 正确写法：

文本表（中文）：
```icu
MSG_FLIGHT_RECORD = 今日飞行距离：{distance, number, ::measure-unit/length-kilometer}，耗时 {time_spent, date, ::mmss}。
```

文本表（英文）：
```icu
MSG_FLIGHT_RECORD = Flight distance today: {distance, number, ::measure-unit/length-mile}, time taken {time_spent, date, ::mmss}.
```
代码传入原始数值和时间：
```csharp
Localization.Get("MSG_FLIGHT_RECORD", new { distance = 12.5, time_spent = flightDuration });
```
> 引擎会自动将 `distance` 格式化为 `12.5` 或 `12,5`（视语言而定），并自动带上 `km` 或 `mile`；时间也会按目标语言的习惯表达。

## 6.5 检查清单

☐ 检查文本表中的占位符，是否有 `{0}`、`{1}` 这种纯数字编号？</br>
☐ 检查代码，是否有用 `+` 号拼接单复数后缀的情况？</br>
☐ 检查代码，是否有根据性别 `if-else` 拼凑代词的情况？</br>
☐ 检查韩语环境，插入玩家自定义名称时，助词是否由程序动态计算？</br>
☐ 检查代码，数字是否直接使用了 `.ToString()` 而没有走本地化格式化 API？</br>
☐ 检查日期和单位，是否直接硬编码了格式符号，而没有交由本地化引擎处理？

## 6.6 参考文献

* Unicode CLDR 项目（通用本地化数据存储库） **[CLDR](https://cldr.unicode.org/)**
* ICU MessageFormat 语法指南 **[ICU Documentation](https://unicode-org.github.io/icu/userguide/format_parse/messages/)**
* Unity 官方文档 - Smart Strings 与语法变体 **[Unity Localization Smart Strings](https://docs.unity3d.com/Packages/com.unity.localization@1.4/manual/Smart/SmartStrings.html)**
* Mozilla 开发者网络 - JavaScript 国际化 API (Intl) **[MDN Intl](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl)**
