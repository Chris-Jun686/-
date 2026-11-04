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

### 6.2.5 数字、日期、时间与货币格式
- **R6-10** 严禁手动截取数字或写死分隔符（如手动加逗号 `,` 或小数点 `.`），必须将原始数值交由本地化引擎自动格式化。
- **R6-11** 严禁在代码或UI中写死日期和时间的格式。日期顺序（如美式 `mm/dd/yyyy`，中式 `yyyy-mm-dd`）以及12/24小时制，必须由文本表或本地化配置决定。
- **R6-12** 货币符号（`$`、`€`、`¥`、`两`）的位置因语言而异（前置或后置），严禁在代码中硬编码拼接，必须由格式化引擎处理。
- **R6-13** 涉及真实货币时，必须根据目标语言/地区的汇率，在代码层或配置表中进行**数值近似换算**，绝不直接将原货币数值套用目标货币符号。
- **R6-14** 序数词（如第1、第2）必须使用专门的语法选择器（如 ICU `selectordinal`），严禁在代码中写死英文的 `st/nd/rd/th` 或其他语言的后缀。

## 6.3 前作发生了什么

### 6.3.1 占位符问题

**问题编号-#06** 

<img width="600" height="300" alt="LQA-006" src="https://github.com/user-attachments/assets/389fb486-549d-4106-9dba-35aa5de9e59f" />

> "__ has", "___'s core" 由于英文未设置变量，主语处空缺

### 6.3.2 单复数问题

**问题编号-#23** 

<img width="600" height="300" alt="LQA-021" src="https://github.com/user-attachments/assets/e5aaee9f-89c6-4837-8f42-e0d0e5879165" />

> "1points" 由于拼接/硬编码，导致缺少空隔以及单复数、语法错误

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

- 错误写法：代码根据性别用 `if-else` 拼凑代词（或者三元运算符）。
```csharp
// 错误示范：将自然语言的逻辑写死在代码里
string pronoun = "";
if (playerGender == Gender.Male) 
{
    pronoun = "他";
} 
else 
{
    pronoun = "她";
}
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

### 6.4.5 数字、日期、时间、货币与序数词格式

**示例：客栈消费账单与江湖悬赏令**

- **中文原文**：大侠于 2023年10月5日 14时30分 在客栈消费，购买上等女儿红 2 坛，共计 15,631.87 两银子。目前你在武林悬赏榜上排名第 3。
- **错误写法**：手动截取数字、硬编码格式和货币符号、强行拼接序数词。
```csharp
// 错误 1：写死日期格式，海外玩家看懵
string dateStr = year + "年" + month + "月" + day + "日 " + hour + "时" + minute + "分";

// 错误 2：写死千分位和小数点（如果到了德语环境，逗号和点应该是反的）
string moneyStr = "$" + amount.ToString("N2"); 

// 错误 3：强行拼写序数词后缀，遇上法语 2e 或日文 3日 直接崩溃
string rankStr = rank + (rank == 1 ? "st" : (rank == 2 ? "nd" : (rank == 3 ? "rd" : "th")));

msg = "大侠于 " + dateStr + " 在客栈消费，共计 " + moneyStr + "。排行榜排名：" + rankStr;
```
- **正确写法**：

**程序层（伪代码）**：
代码只负责传原始数据和换算后的数值，由本地化库处理格式。
```csharp
Localization.Get("MSG_INN_BILL", new { 
    date = DateTime.Now,     
    item = "女儿红",
    count = 2,
    amount = convertedPrice, // 换算后的本地化货币数值
    rank = 3                 // 传入整数排名
});
```

**文本表配置**：

中文文本表 (zh-CN)：
```icu
MSG_INN_BILL = 大侠于 {date, date, medium} {date, time, short} 在客栈消费，购买{item} {count} 坛，共计 {amount, number, ::currency/CNY}。目前你在武林悬赏榜上排名第 {rank, selectordinal, other {#}}。
```
*(中文引擎渲染结果：...共计 ¥15,631.87。目前你在武林悬赏榜上排名第 3。)*

英文文本表 (en-US)：
```icu
MSG_INN_BILL = You spent {amount, number, ::currency/USD} on {count} {item} at the inn on {date, date, short} at {date, time, short}. You are currently ranked {rank, selectordinal, one {#st} two {#nd} few {#rd} other {#th}} on the martial arts bounty board.
```
*(英文引擎渲染结果：...on 10/5/23 at 2:30 PM. You are currently ranked 3rd... **注意：日期格式变为 mm/dd/yyyy，使用12小时制，货币前置且被换算为美元，序数词自动变为 3rd。**)*

日文文本表 (ja-JP)：
```icu
MSG_INN_BILL = {date, date, ::yyyy年MM月dd日} {date, time, ::HH:mm} に宿屋で{item}を{count}本購入し、合計 {amount, number, ::currency/JPY} を支払いました。現在の武林懸賞榜の順位は {rank, selectordinal, other {#位}} です。
```
*(日文引擎渲染结果：...合計 ￥1,250,550 を支払いました。...順位は 3位 です。 **注意：日期变为 yyyy年MM月dd日，24小时制，货币后置，序数词使用日式“位”。**)*

## 6.5 检查清单

☐ 检查文本表中的占位符，是否有 `{0}`、`{1}` 这种纯数字编号？</br>
☐ 检查代码，是否有用 `+` 号拼接单复数后缀的情况？</br>
☐ 检查代码，是否有根据性别 `if-else` 拼凑代词的情况？</br>
☐ 检查韩语环境，插入玩家自定义名称时，助词是否由程序动态计算？</br>
☐ 检查代码，数字是否直接使用了 `.ToString()` 而没有走本地化格式化 API？</br>
☐ 检查日期、时间、货币符号，是否直接硬编码了格式，而没有交由本地化引擎处理？</br>
☐ 检查序数词（如第几名），是否使用了 `selectordinal` 等引擎特性，而非手动拼接后缀？</br>
☐ 如果游戏涉及真实货币换算，检查代码或配置表中是否处理了汇率？

## 6.6 参考文献

* Unicode CLDR 项目（通用本地化数据存储库） **[CLDR](https://cldr.unicode.org/)**
* ICU MessageFormat 语法指南 **[ICU Documentation](https://unicode-org.github.io/icu/userguide/format_parse/messages/)**
* Unity 官方文档 - Smart Strings 与语法变体 **[Unity Localization Smart Strings](https://docs.unity3d.com/Packages/com.unity.localization@1.4/manual/Smart/SmartStrings.html)**
* Mozilla 开发者网络 - JavaScript 国际化 API (Intl) **[MDN Intl](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl)**
