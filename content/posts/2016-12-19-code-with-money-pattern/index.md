---
title: "ให้เงินทำงานด้วย Money Pattern"
date: 2016-12-19T12:00:00+07:00
tags: ["tech", "programming", "php"]
language: "thai"
---

_เผยแพร่ครั้งแรกที่ [blog.maqe.com](https://blog.maqe.com/why-code-with-money-pattern-5d5b98c3252d)_

{{<figure src="resources/01-money-php.png">}}

> "เอ้า ตังค์ทอน 100 บาท มากัน 3 คน … ฉีกแบงค์กันไปคนละส่วนแล้วกัน"

ประโยคนี้น่าจะถูกใช้เป็นมุกบนโต๊ะอาหารอยู่บ่อย ๆ แต่ถ้าเหตุการณ์นี้เกิดขึ้นในระบบที่เราต้องให้ความมั่นใจผู้ใช้งานได้ ว่าเงินทุกบาททุกสตางค์ถูกคำนวนมาอย่างถูกต้อง เที่ยงตรง ไม่มีการมุมมิบ เราจะยังขำกันได้อยู่หรือเปล่า?

ในยุคที่การให้เหรียญสลึง = แช่ง การคำนวนเศษในหลักสตางค์ก็คงเป็นเรื่องที่ไม่สำคัญนัก หากแต่ในระบบที่มีคนใช้เป็นหมื่นเป็นแสนคน มีการทำรายการนับครั้งไม่ถ้วนต่อวัน ปัญหาของเศษเสี้ยวสตางค์จะกลายเป็นปัญหาระดับร้อยล้านพันล้านไปในทันที ค่าที่ผิดไปเพียง 0.01 บาทจาก 1 แสนรายการต่อวัน คิดเป็นเงินกว่า 350,000 บาทต่อปี เงินจำนวนนี้ไปตกอยู่ที่ไหน เราสามารถปล่อยให้มันหายไปในอากาศได้หรือเปล่า?

ถ้าคำตอบคือจะยอมให้หายไปไม่ได้ … Money Pattern (หรือ Money Object, Money Value ฯลฯ) จึงเป็นหนึ่งใน Design Pattern ที่ควรพกติดตัวไว้ เพราะในสมัยนี้ คงระบบที่จะไม่ได้ยุ่งกับจำนวนเงินเลยน้อยลงเรื่อย ๆ ทุกวัน

และถึงแม้ว่าเราจะไม่มีโอกาสได้ทำงานกับจำนวนเงินเลย ปัญหานี้ก็เป็นตัวอย่างเตือนสติได้อย่างดี ว่าการทำงานกับระบบคอมพิวเตอร์และการเขียนลอจิคให้ครอบคลุมการใช้งานของมนุษย์นั้น มันไม่ได้ง่ายเหมือนจิ้มเครื่องคิดเลขบวกลบคูณหารทีเดียวจบเสมอไป

---

## ปัญหาของการเขียนโค้ดกับจำนวนเงิน

### ปัญหาที่ 1: การแบ่งเงินเป็นกองๆ … มีเศษที่หายไป

มีเงินอยู่ 10,000 บาท จะแบ่งฝากเข้าบัญชี 3 บัญชีเท่า ๆ กัน (บัญชีเงินลงทุน, บัญชีเงินเก็บไปเที่ยว, บัญชีเงินค่าขนม) จะต้องฝากบัญชีละเท่าไหร่?

คำตอบก็คือเอา 10,000 หาร 3 แล้วโอนเข้าไป? นี่คือคำตอบที่… ผิด

มันไม่ได้ง่ายอย่างนั้น เพราะ 10,000 / 3 = 3,333.3333333 เป็นเลขที่หารไม่ลงตัว และการตัดเหลือ 3,333.33 บาท ก็จะทำให้ผลรวมขาดไป 0.01 บาท เงินจำนวนนี้จะไม่มีที่ลงไม่ได้

### ปัญหาที่ 2: สกุลเงินต่างกัน … จะนำมาคำนวนกันโดยตรงไม่ได้

100 USD + 100 THB จะต้องใช้อัตราแลกเปลี่ยนอะไร? ผลลัพธ์จะต้องเป็น USD หรือ THB? คำถามนี้ให้เห็นว่า 100 USD + 100 THB ต้องมีข้อมูลมากกว่าแค่จำนวนเงินสองตัวในการคำนวน

เราจะการันตีได้อย่างไรว่าเราได้ป้องกันไม่ให้เกิดการคำนวนเงินข้ามสกุลเงินตรง ๆ ไว้แล้ว? และการต้องเขียนโค้ดให้ดักสกุลเงินทุกรอบที่มีการคำนวน ใช่คำตอบที่ดีที่สุดหรือเปล่า?

### ปัญหาที่ 3: คอมพิวเตอร์มองจุดทศนิยมไม่เหมือนมนุษย์

ลองเปิด Developer Console แล้วลองเลย…

```js
> 0.10 + 0.20 == 0.30
> false
```

เนื่องจากคอมพิวเตอร์เก็บเลขทศนิยมเป็น float มันจึงไม่สามารถเก็บและคำนวนได้อย่างแม่นยำเสมอไป (ชาว stackoverflow มีอธิบายไว้อย่างละเอียด) ซึ่งยิ่งมีการผลลัพธ์ไปคำนวนต่อมากเท่าไหร่ ก็จะยิ่งทำให้จำนวนเงินบิดเบือนไปจากจำนวนที่ควรจะเป็นมากขึ้นเรื่อยๆ

และที่สำคัญที่สุดก็คือ ปัญหานี้ไม่ได้ขึ้นอยู่กับภาษาโปรแกรมมิ่งภาษาใดภาษาหนึ่ง แต่เป็นปัญหาพื้นฐานที่จะเกิดขึ้นตราบใดที่เรายังคงต้องใช้คอมพิวเตอร์ในการทำงานกับจำนวนเงิน และ/หรือเลขทศนิยม

{{<figure src="resources/02-js-console.png">}}

---

## คำตอบ: Money Pattern ช่วยคุณได้

[Money Pattern](https://martinfowler.com/eaaCatalog/money.html) เป็น Design Pattern สำหรับเก็บและทำงานกับจำนวนเงินในโค้ด ซึ่งถูกนิยามไว้ในหนังสือ [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html) โดย Martin Fowler โดยเขากล่าวถึงปัญหาไว้ว่า

> Once you involve multiple currencies you want to avoid adding your dollars to your yen without taking the currency differences into account.
>
> The more subtle problem is with rounding. Monetary calculations are often rounded to the smallest currency unit. When you do this it's easy to lose pennies (or your local equivalent) because of rounding errors.
>
> The good thing about object-oriented programming is that you can fix these problems by creating a Money class that handles them.

เนื่องจากผมได้เข้าไป [contribute ในโปรเจกต์นี้](https://github.com/moneyphp/money/pulls?q=author%3Aunnawut)อยู่บ้าง จึงขอยก [moneyphp/money](https://github.com/moneyphp/money) มาเป็นตัวอย่าง…

แนวทางคร่าว ๆ ก็คือ การจับจำนวนเงินทั้งหมดให้อยู่ในรูปแบบ Money objects แล้วทำงานกับ objects เหล่านี้ แทนที่จะทำงานกับตัวเลขโดยตรง เช่น

```php
<?php
$cash = Money::THB(100);  // ฿1.00
```

ตัว Money object จะมีตัวช่วยหลาย ๆ อย่างที่ทำให้การทำงานสะดวกยิ่งขึ้น เช่น `add()`, `subtract()`, `allocateTo()`, `equals()`, `greaterThan()` ฯลฯ ที่จะช่วยให้เราจัดการกับปัญหาที่กล่าวถึงก่อนหน้านี้ได้ง่ายขึ้นมาก ๆ

ข้อควรระวัง: เนื่องจากการส่งค่าจุดทศนิยมมีโอกาสทำให้ค่าที่รับเข้าไปผิดตั้งแต่แรก (garbage in, garbage out) การใช้ Money pattern จึงยึดค่าโดยใช้หน่วยเล็กที่สุดของสกุลเงินนั้นเสมอ เช่น 1 บาทก็ต้องใส่ `Money::THB(100)`

---

### ปัญหาที่ 1: การแบ่งเงินเป็นกองๆ … มีเศษที่หายไป

ใช้ `allocateTo()` ในการแบ่งเงินออกเป็นกอง ๆ จำนวนเท่า ๆ กัน เศษของเงินจะถูกกระจายออกให้ได้มากที่สุด ทำให้ผลรวมของการแบ่งเท่ากับจำนวนเงินตั้งต้น เช่น

```php
<?php
$profit = Money::THB(1000000);  // ฿10000.00
$profit->allocateTo(3);         // [฿3333.34, ฿3333.33, ฿3333.33]
```

การแบ่งเงินออกเป็นหลาย ๆ สัดส่วนด้วย `allocate()`:

```php
<?php
$profit = Money::THB(500);    // ฿5.00
$profit->allocate([70, 30]);  // [฿4.00, ฿1.00]
```

การเรียงสัดส่วนไม่เหมือนกันก็มีผลต่อการแบ่ง:

```php
<?php
$profit = Money::THB(500);    // ฿5.00
$profit->allocate([30, 70]);  // [฿2.00, ฿3.00]
```

### ปัญหาที่ 2: สกุลเงินต่างกัน … จะนำมาคำนวนกันโดยตรงไม่ได้

เช็คสกุลเงินก่อนทำการคำนวน เพื่อป้องกันไม่ให้มีการคำนวนข้ามสกุลเงินโดยไม่ตั้งใจ:

```php
<?php
$cashTHB = Money::THB(1000);  // ฿1.00
$cashUSD = Money::USD(1000);  // $1.00

$cashTHB->add($cashUSD);      // Exception
```

### ปัญหาที่ 3: คอมพิวเตอร์มองจุดทศนิยมไม่เหมือนมนุษย์

ผู้พัฒนา Money object แต่ละตัวจะเลือกเก็บจำนวนเงินโดยไม่ใช้ float หรือ double เพื่อหลีกเลี่ยงปัญหาในการคำนวนกับ floating points ตัวอย่างที่เกิดขึ้นใน Developer Tool ด้านบนก็จะไม่เกิดขึ้นเมื่อใช้ Money object

```php
<?php
$myCash = Money::THB(1);             // ฿0.01
$yourCash = Money::THB(2);           // ฿0.02

$ourCash = $myCash->add($yourCash);  // ฿0.03
```

### ข้อควรระวังในการใช้ Money object

- Money object จะรับค่าเป็นหน่วยย่อยที่สุด เช่น Money::THB(100) = 1 บาท (100 สตางค์) ไม่ใช่ 100 บาท
- Money object เป็น immutable object เสมอ เช่น

```php
<?php
$original = Money::THB(10000)       // ฿100.00
$discount = Money::THB(1000)        //  ฿10.00

$salePrice = $original->subtract($discount)  //  ฿90.00
$originalPrice                      // ฿100.00
```

หากไม่ได้เขียนโค้ดมาครอบคลุมพอ การเรียก `$originalPrice->subtract()` จะทำให้ $originalPriceเปลี่ยนไปด้วย​ ซึ่งสิ่งที่ควรเปลี่ยนคือ $salePrice ไม่ใช่ $originalPrice


---

## Money Pattern ในภาษาต่าง ๆ

- PHP: https://github.com/moneyphp/money
- Ruby: https://github.com/RubyMoney/money
- Python: https://code.google.com/archive/p/python-money
- Javascript: https://github.com/openexchangerates/money.js
- Go: https://github.com/leekchan/accounting
- Swift: https://github.com/danthorpe/Money
- Java: https://github.com/JavaMoney/

แน่นอนว่า **Money Pattern ไม่ใช่ silver bullet** ที่สามารถการันตีได้ว่านักพัฒนาจะไม่ต้องสนใจการทำงานกับจำนวนเงินอีกเลย แต่การใช้ Money Pattern ก็ช่วยให้เราควบคุมพฤติกรรมของการคำนวนได้ดีขึ้น โดยเป็นการบังคับให้โค้ดที่ทำงานกับเงินจะต้องทำผ่านฟังค์ชั่นที่เรากำหนดไว้ มีพฤติกรรมที่ชัดเจน และเมื่อมีการทำงานที่ต้องห้าม ระบบก็จะสามารถฟ้องเราได้ทันที

รู้อย่างนี้แล้ว พร้อมให้เงินทำงาน (ไปกับโค้ดของเรา) แล้วหรือยัง?


---

บทความนี้เป็นประสบการณ์ส่วนหนึ่งจากการร่วมโปรเจกต์กับพี่หมี [aimakun](https://medium.com/@aimakun) ที่บริษัท MAQE Bangkok


---

ปล.1 อ่านเพิ่มเติม ทำไมเราไม่ควรใช้ float หรือ double ในการเก็บจำนวนเงิน [Why not use Double or Float to represent currency?](https://stackoverflow.com/questions/3730019/why-not-use-double-or-float-to-represent-currency)

ปล.2 อ่านเวอร์ชั่นยาวและละเอียดยิบได้ที่ [What Every Computer Scientist Should Know About Floating-Point Arithmetic](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)

ปล.3 ใน PHP ยังใช้เครื่องหมายบวกลบคูณหารกับ money object แบบ Ruby ไม่ได้ ต้องใช้ `->add()`, `->subtract()` ฯลฯ ? รอหน่อยพี่ [operator overloading ยังไม่มา](https://wiki.php.net/rfc/operator-overloading)

ปล.4 อ่านการพิสูจน์ทางคณิตศาสตร์ของ money allocation ได้ที่ [Proof that Fowler's money allocation algorithm is correct](https://stackoverflow.com/questions/1679292/proof-that-fowlers-money-allocation-algorithm-is-correct)
