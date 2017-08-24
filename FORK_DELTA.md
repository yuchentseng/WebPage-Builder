---
layout: docs
title: Designing the Delta Format
fork: quill/docs/guides/designing-the-delta-format/
---

Rich text editors lack a specification to express its own contents. Until recently, most rich text editors did not even know what was in their own edit areas. These editors just pass the user HTML, along with the burden of parsing and interpretting this. At any given time, this interpretation will differ from those of major browser vendors, leading to different editing experiences for users.
富文本編輯器缺乏一個描述本身內容的規範。 直到最近，大多數富文本編輯不知道自己的編輯區域內有些什麼。 這些編輯器只是傳遞用戶HTML，以及一大堆的解析和解釋。這種解釋方式將與主要瀏覽器廠商的不同，導致用戶的不同編輯體驗。

Quill is the first rich text editor to actually understand its own contents. Key to this is Deltas, the specification describing rich text. Deltas are designed to be easy to understand and use. We will walk through some of the thinking behind Deltas, to shed light on *why* things are the way they are.
Quill是第一個實際了解自己內容的富文本編輯器。描述富文本規範的關鍵就是Deltas。 Deltas設計易於理解和使用。 我們將走過Deltas背後的思考，揭示*為什麼*他是如此樣貌。

If you are looking for a reference on *what* Deltas are, the [Delta documentation](/docs/delta/) is a more concise resource.
如果您是想知道*什麼*是Deltas，參考[Delta文檔]（/ docs / delta /）是一個更簡潔的資源。


## Plain Text
## 純文字

Let's start at the basics with just plain text. There already is a ubiquitous format to store plain text: the string. Now if we want to build upon this and describe formatted text, such as when a range is bold, we need to add additional information.
讓我們從簡單的純文字開始。 已經有一個普遍存在的格式來存儲純文字：字符串。 現在，如果我們想建立這個並描述格式化的文本，例如描述某些段落是粗體時，我們需要添加其他信息。

Arrays are the only other ordered data type available, so we use an array of objects. This also allows us to leverage JSON for compatibility with a breadth of tools.
陣列是唯一有序的數據類型，因此我們使用一個物件陣列。這也使我們能夠利用JSON來兼容其他泛用的工具。

```javascript
var content = [
  { text: 'Hello' },
  { text: 'World', bold: true }
];
```

We can add italics, underline, and other formats to the main object if we want to; but it is cleaner to separate `text` from all of this so we organize formatting under one field, which we will name `attributes`.
我們可以在主要物件中添加斜體，下劃線和其他格式; 但將`text`獨立出來更為簡潔，所以我們將格式標註在另一個field，命名為`attributes`。

```javascript
var content = [
  { text: 'Hello' },
  { text: 'World', attributes: { bold: true } }
];
```


### Compact
### 緊湊

Even with our simple Delta format so far, it is unpredictable since the above "Hello World" example can be represented differently, so we cannot predict which will be generated:
目前為止，即使簡單如Delta，其格式也是不可預知的，因為上述“Hello World”的例子可以用不同的方式表示，所以我們無法預測會產生哪一個：

```javascript
var content = [
  { text: 'Hel' },
  { text: 'lo' },
  { text: 'World', attributes: { bold: true } }
];
```

To solve this, we add the constraint that Deltas must be compact. With this constraint, the above representation is not a valid Delta, since it can be represented more compactly by the previous example, where "Hel" and "lo" were not separate. Similarly we cannot have `{ bold: false, italic: true, underline: null }`, because `{ italic: true }` is more compact.
為了解決這個問題，我們添加了Deltas必須緊湊的約束。 有了這個約束，上面的表示法不是一個有效的Delta，因為它可以被前面的例子更緊湊地表示，其中“Hel”和“lo”不是分開的。 類似地，我們不能使用`{bold：false，italic：true，underline：null}`，因為`{italic：true}`更加緊湊。


### Canonical
### 規範的

We have not assigned any meaning to `bold`, just that it describes some formatting for text. We could very well have used different names, such as `weighted` or `strong`, or used a different range of possible values, such as a numerical or descriptive range of weights. An example can be found in CSS, where most of these ambiguities are at play. If we saw bolded text on a page, we cannot predict if its rule set is `font-weight: bold` or `font-weight: 700`. This makes the task of parsing CSS to discern its meaning, much more complex.
我們沒有給“bold”指定任何含義，只是它描述了一些文字的格式。 我們可以非常好地使用不同的名稱，例如“weighted”或“strong”，或者使用不同範圍的可能值，例如數值或權重範圍描述。 CSS中可以找到一個例子，其中有許多同樣效果的表達式。 如果我們在頁面上看到粗體文本，我們無法預測其規則集是“font-weight：bold”還是“font-weight：700”。 這使得解析CSS來辨別其含義的任務，更為複雜。

We do not define the set of possible attributes, nor their meanings, but we do add an additional constraint that Deltas must be canonical. If two Deltas are equal, the content they represent must be equal, and there cannot be two unequal Deltas that represent the same content. Programmatically, this allows you to simply deep compare two Deltas to determine if the content they represent is equal.
我們不定義一組可能的屬性，也不定義它們的含義，但是我們添加了一個額外的限制，即Deltas必須是規範的。 如果兩個Deltas相等，它們所代表的內容必須相等，並且不能有兩個不等的Deltas表示相同的內容。 以編程方式，您可以簡單地深入比較兩個Deltas，以確定它們所代表的內容是否相等。

So if we had the following, the only conclusion we can draw is `a` is different from `b`, but not what `a` or `b` means.
以下的例子來說，我們可以得出唯一結論是`a`與`b`是不同的，但不知道`a`或`b`的代表的含義。

```javascript
var content = [{
  text: "Mystery",
  attributes: {
    a: true,
    b: true
  }
];
```

It is up to the implementer to pick appropriate names:
實作時應該選擇適當的名稱：

```javascript
var content = [{
  text: "Mystery",
  attributes: {
    italic: true,
    bold: true
  }
];
```

This canonicalization applies to both keys and values, `text` and `attributes`. For example, Quill by default:
這個規範化適用於鍵和值，`text`和`attributes`。 例如，Quill默認情況下：

- Uses six character hex values to represent colors and not RGB
- There is only one way to represent a newline which is with `\n`, not `\r` or `\r\n`
- <code>text: "Hello&nbsp;&nbsp;World"</code> unambiguously means there are precisely two spaces between "Hello" and "World"
- 使用六個字符的十六進制值來表示顏色而不是RGB
- 只有一種方法來表示一個換行符，它與`\ n`而不是`\ r`或`\ r \ n`
- <code> text：“Hello＆nbsp;＆nbsp; World”</ code>明確地表示“Hello”和“World”之間正好有兩個空格

Some of these choices may be customized by the user, but the canonical constraint in Deltas dictate that the choice must be unique.
其中如何表示可以由用戶定制，但Deltas中的規範約束規定了表示必須是唯一的。

This unambiguous predictability makes Deltas easier to work with, both because you have fewer cases to handle and because there are no surprises in what a corresponding Delta will look like. Long term, this makes applications using Deltas easier to understand and maintain.
這個明確的可預測性使得Deltas更容易使用，因為您處理的案例較少，相應的Delta也不會有任何超乎預期的地方。 長期來看，這使得使用Deltas的應用程序更容易理解和維護。


## Line Formatting
## 行格式

Line formats affect the contents of an entire line, so they present an interesting challenge for our compact and canonical constraints. A seemingly reasonable way to represent centered text would be the following:
行格式影響整行的內容，這點對我們的緊湊和規範的約束提出了一個有趣的挑戰。 一個看似合理的表示置中對齊的方法如下：

```javascript
var content = [
  { text: "Hello", attributes: { align: "center" } },
  { text: "\nWorld" }
];
```

But what if the user deletes the newline character? If we just naively get rid of the newline character, the Delta would now look like this:
但是如果用戶刪除換行符呢？ 如果我們只是簡單地去除換行符號，那麼Delta現在看起來就像這樣：

```javascript
var content = [
  { text: "Hello", attributes: { align: "center" } },
  { text: "World" }
];
```

Is this line still centered? If the answer is no, then our representation is not compact, since we do not need the attribute object and can combine the two strings:
那麼這行文字是否還是置中呢？ 如果答案為否，那麼我們的表示並不緊湊，因為我們不需要屬性物件並且可以組合兩個字符串：

```javascript
var content = [
  { text: "HelloWorld" }
];
```

But if the answer is yes, then we violate the canonical constraint since any permutation of characters having an align attribute would represent the same content.
但是如果答案是肯定的，那麼我們違反規範約束，因為只要具有對齊屬性的字符的排列將表示相同的內容。

So we cannot just naively get rid of the newline character. We also have to either get rid of line attributes, or expand them to fill all characters on the line.
所以我們不能直接去除換行符號。 我們還必須去除行屬性，或者將屬性補上同一行上的所有字符。


What if we removed the newline from the following?
如果我們從以下內容中刪除換行呢？

```javascript
var content = [
  { text: "Hello", attributes: { align: "center" } },
  { text: "\n" },
  { text: "World", attributes: { align: "right" } }
];
```

It is not clear if our resulting line is aligned center or right. We could delete both or have some ordering rule to favor one over the other, but our Delta is becoming more complex and harder to work with on this path.
無法定義這個行是置中還是靠右對齊。 我們可以將兩者都刪除，或是制定順序規則只保留其中一個，但是如此Delta將變得越來越複雜和越來越難。

This problem begs for atomicity, and we find this in the *newline* character itself. But we have an off by one problem in that if we have *n* lines, we only have *n-1* newline characters.
這個問題需要原子性，* newline *字符本身中就具有原子性。 但是我們有一個問題，如果我們有* n *行，我們只有* n-1 *個換行符。

To solve this, Quill "adds" a newline to all documents and always ends Deltas with "\n".
為了解決這個問題，Quill“在所有文檔的最後添加了一個換行符，Deltas始終以”\ n“結尾。

```javascript
// Hello World on two lines
var content = [
  { text: "Hello" },
  { text: "\n", attributes: { align: "center" } },
  { text: "World" },
  { text: "\n", attributes: { align: "right" } }   // Deltas must end with newline
];
```


## Embedded Content

We want to add embedded content like images or video. Strings were natural to use for text but we have a lot more options for embeds. Since there are different types of embeds, our choice just needs to include this type information, and then the actual content. There are many reasonable options here but we will use an object whose only key is the embed type and the value is the content representation, which may have any type or value.
我們要添加圖像或視頻等嵌入式內容。文本想當然爾有字符串，但是我們有其他更多的嵌入選項。由於有不同類型的嵌入，我們的選擇只需要包括這種類型的信息，然後是實際的內容。 這裡有許多合理的作法，但是我們將唯一使用一種物件，其鍵值分別為：任何形式的嵌入類型；任何形式的內容值。

```javascript
var img = {
  image: {
    url: 'https://quilljs.com/logo.png'
  }
};

var f = {
  formula: 'e=mc^2'
};
```

Similar to text, images might have some defining characteristics, and some transient ones. We used `attributes` for text content and can use the same `attributes` field for images. But because of this, we can keep the general structure we have been using, but should rename our `text` key into something more general. For reasons we will explore later, we will choose the name `insert`. Putting this all together we have:
與文本類似，圖像可能具有一些定義特徵，也可能具有一些臨時特徵。 我們可以對圖像使用“attributes”字段如同使用`attributes`表示文本內容。 但是正因為如此，我們可以保持我們一直在使用的一般結構，而應該將我們的`text`鍵重命名為更通用的東西。 為了我們稍後再探討，我們將選擇'insert`。 把這一切放在一起我們有：

```javascript
var content = [{
  insert: 'Hello'
}, {
  insert: 'World',
  attributes: { bold: true }
}, {
  insert: {
    image: 'https://exclamation.com/mark.png'
  },
  attributes: { width: '100' }
}];
```


## Describing Changes
## 描述變化

As the name Delta implies, our format can describe changes to documents, as well as documents themselves. In fact we can think of documents as the changes we would make to the empty document, to get to the one we are describing. As you might have already guessed, using Deltas to also describe changes is why we renamed `text` to `insert` earlier. We call each element in our Delta array an Operation.
正如Delta名稱中所暗示的，我們的格式可以描述文檔的更改以及文檔本身。 事實上，我們可以將文檔視為對空白文檔的更改，以獲得我們正在描述的文檔。 正如您可能已經猜到的，使用Deltas來描述更改是為什麼我們將`text`更改為`insert`的原因。 我們調用我們的Delta數組中的每個元素一個操作。

#### Delete
#### 刪除

To describe deleting text, we need to know where and how many characters to delete. To delete embeds, there needs not be any special treatment, other than to understand the length of an embed. If it is anything other than one, we would then need to specify what happens when only part of an embed is deleted. There is currently no such specification, so regardless of how many pixels make up an image, how many minutes long a video is, or how many slides are in a deck; embeds are all of length **one**.
為了描述刪除文本，我們需要知道要刪除的字符在哪里和刪除多少。要刪除嵌入，除了了解嵌入的長度之外，不需要任何特殊的處理。 如果刪除的嵌入不是一整個，那麼我們需要指定當只有一部分嵌入被刪除時會發生什麼。 目前沒有這樣的規範，所以不管構成圖像的像素多少，視頻有多少分鐘，或是甲板上有多少張幻燈片？ 嵌入都是**一個**。

One reasonable way to describe a deletion is to explicitly store its index and length.
描述刪除的一種合理方法是明確標註其索引和長度。

```javascript
var delta = [{
  delete: {
    index: 4,
    length: 1
  }
}, {
  delete: {
    index: 12,
    length: 3
  }
}];
```

We would have to order the deletions based on indexes, and ensure no ranges overlap, otherwise our canonical constraint would be violated. There are a couple other shortcomings to this index and length approach, but they are easier to appreciate after describing format changes.
我們必鬚根據索引排序刪除，並確保沒有範圍重疊，否則違反規範約束。 這種索引和長度方法還有一些其他缺點，但是在描述格式更改後更容易理解。

#### Insert
#### 插入

Now that Deltas may be describing changes to a non-empty document, `{ insert: "Hello" }` is insufficient, because we do not know where "Hello" should be inserted. We can solve this by also adding an index, similar to `delete`.
假設現在Deltas可能正在描述對非空文檔的更改，`{insert：“Hello”}`是不夠的，因為我們不知道要在何處插入“Hello”。 我們可以通過添加一個索引來解決這個問題，類似於`delete`。

#### Format
#### 格式

Similar to deletes, we need to specify the range of text to format, along with the format change itself. Formatting exists in the `attributes` object, so a simple solution is to provide an additional `attributes` object to merge with the existing one. This merge is shallow to keep things simple. We have not found a use case that is compelling enough to require a deep merge and warrants the added complexity.
與刪除類似，我們需要指定要格式化的文本範圍，以及格式更改本身。“attributes”物件本是描述格式化，因此一個簡單的解決方案是提供一個附加的`attributes`物件與現有物件進行合併。 這個合併很淺以保持簡單。 我們還沒有找到一個足夠強大的用例來進行深度合併，並且增加了複雜性。

```javascript
var delta = [{
  format: {
    index: 4,
    length: 1
  },
  attributes: {
    bold: true
  }
}];
```

The special case is when we want to remove formatting. We will use `null` for this purpose, so `{ bold: null }` would mean remove the bold format. We could have specified any falsy value, but there may be legitimate use cases for an attribute value to be `0` or the empty string.
特殊情況是我們想要刪除格式。 我們將使用`null`作為此目的，所以`{bold：null}`意味著刪除粗體格式。 我們可以指定任何偽造值，但屬性值可能是合法的用例為0或空字符串。

**Note:** We now have to be careful with indexes at the application layer. As mentioned earlier, Deltas do not ascribe any inherent meaning to any the `attributes`' key-value pairs, nor any embed types or values. Deltas do not know an image does not have duration, text does not have alternative texts, and videos cannot be bolded. The following is a *legal* Delta that might have been the result of applying other *legal* Deltas, by an application not being careful of format ranges.
**注意：**我們現在必須小心應用層上的索引。 如前所述，Deltas並不對任何“屬性”鍵值對以及任何嵌入類型或值賦予任何固有的含義。 Deltas不知道圖像沒有持續時間，文本沒有替代文字，視頻不能加粗。 以下是應用時不小心格式範圍所造成的*合法* Delta可能是應用其他*合法* Deltas的結果。

```javascript
var delta = [{
  insert: {
    image: "https://imgur.com/"
  },
  attributes: {
    duration: 600
  }
}, {
  insert: "Hello",
  attributes: {
    alt: "Funny cat photo"
  }
}, {
  insert: {
    video: "https://youtube.com/"
  },
  attributes: {
    bold: true
  }
}];
```

#### Pitfalls
#### 陷阱

First, we should be clear that an index must refer to its position in the document **before** any Operations are applied. Otherwise, a later Operation may delete a previous insert, unformat a previous format, etc., which would violate compactness.
首先，在任何指定操作之前，我們應該明確*先指出索引位置*。 否則，稍後的操作可以刪除先前的插入，取消格式化先前的格式等，這將違反緊湊性。

Operations must also be strictly ordered to satisfy our canonical constraint. Ordering by index, then length, and then type is one valid way this can be accomplished.
操作也必須嚴格按照我們的規範限制。一種可以實現的有效方法是先按索引排序，然後按長度排序，最後是型態。

As stated earlier, delete ranges cannot overlap. The case against overlapping format ranges is less brief, but it turns out we do not want overlapping formats either.
如前所述，刪除範圍不能重疊。 對於重疊格式範圍的情況不太簡單，但事實證明，我們也不希望重疊格式。

The number of reasons a Delta might be invalid is piling up. A better format would simply not allow such cases to be expressed at all.
Delta可能無效的原因正在堆積。 更好的格式根本不允許這樣的案件被表達出來。

#### Retain
#### 保留

If we step back from our compactness formalities for a moment, we can describe a much simpler format to express inserting, deleting, and formatting:
如果暫時不管格式必須緊湊這點，我們可以描述一種更簡單的格式來表示插入，刪除和格式化：

- A Delta would have Operations that are at least as long as the document being modified.
- Each Operation would describe what happens to the character at that index.
- Optional insert Operations may make the Delta longer than the document it describes.
- Delta將具有至少與正在修改的文檔一樣長的操作。
- 每個操作將描述該索引處的字符發生了什麼。
- 可選插入操作可能使Delta長於其描述的文檔。

This necessitates the creation of a new Operation, that will simply mean "keep this character as is". We call this a `retain`.
這需要創建一個新的操作，就是簡單地“保持這個字符原來的樣子”。 我們稱之為“保留”。

```javascript
// Starting with "HelloWorld",
// bold "Hello", and insert a space right after it
var change = [
  { format: true, attributes: { bold: true } },  // H
  { format: true, attributes: { bold: true } },  // e
  { format: true, attributes: { bold: true } },  // l
  { format: true, attributes: { bold: true } },  // l
  { format: true, attributes: { bold: true } },  // o
  { insert: ' ' },
  { retain: true },  // W
  { retain: true },  // o
  { retain: true },  // r
  { retain: true },  // l
  { retain: true }   // d
]
```

Since every character is described, explicit indexes and lengths are no longer necessary. This makes overlapping ranges and out-of-order indexes impossible to express.
由於每個字符都被描述了，因此不再需要明確的索引和長度。 這使得重疊範圍和超出的索引無法被表達。

Therefore, we can make the easy optimization to merge adjacent equal Operations, re-introducing *length*. If the last Operation is a `retain` we can simply drop it, for it simply instructs to "do nothing to the rest of the document".
因此，我們可以做個簡單的優化，合併相鄰且相等操作，重新引入* length *。 如果最後一個操作是`retain`，我們可以簡單地刪除它，因為它指示“對文檔的其餘部分什麼都不做”。

```javascript
var change = [
  { format: 5, attributes: { bold: true } }
  { insert: ' ' }
]
```

Furthermore, you might notice that a `retain` is in some ways just a special case of `format`. For instance, there is no practical difference between `{ format: 1, attributes: {} }` and `{ retain: 1 }`. Compacting would drop the empty `attributes` object leaving us with just `{ format: 1 }`, creating a canonicalization conflict. Thus, in our example we will simply combine `format` and `retain`, and keep the name `retain`.
此外，您可能會注意到`retain`在某些方面只是格式化的一種特殊情況。 例如，`{format：1，attributes：{}}`和`{retain：1}`之間沒有實際的區別。 壓縮將刪除空的`attributes`對象，只留下'{format：1}`，創建規範化衝突。 因此，在我們的示例中，我們將簡單地合併“format”和“retain”，只保留名稱“retain”。

```javascript
var change = [
  { retain: 5, attributes: { bold: true } },
  { insert: ' ' }
]
```

We now have a Delta that is very close to the current standard format.
我們現在有一個與目前標準格式非常接近的Delta。

#### Ops

Right now we have an easy to use JSON Array that describes rich text. This is great at the storage and transport layers, but applications could benefit from more functionality. We can add this by implementing Deltas as a class, that can be easily initialized from or exported to JSON, and then providing it with relevant methods.
現在我們有一個易於使用的JSON數組描述豐富的文本。 這在存儲和傳輸層是非常好的，但應用程序可以從更多的功能中受益。 我們可以通過將Deltas實現為一個類來添加，可以輕鬆地從JSON初始化或導出到JSON，然後為其提供相關的方法。

At the time of Delta's inception, it was not possible to sub-class an Array. For this reason Deltas are expressed as Objects, with a single property `ops` that stores an array of Operations like the ones we have been discussing.
在Delta成立時，不可能對數組進行子類化。 因此，我們用物件來表示Deltas，具有單個屬性“ops”，存儲操作數組，就像我們一直在討論的那樣。

```javascript
var delta = {
  ops: [{
    insert: 'Hello'
  }, {
    insert: 'World',
    attributes: { bold: true }
  }, {
    insert: {
    image: 'https://exclamation.com/mark.png'
    },
    attributes: { width: '100' }
  }]
};
```

Finally, we arrive at the [Delta format](/docs/delta/), as it exists today.
