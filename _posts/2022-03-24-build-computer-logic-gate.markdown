---
layout: post
title: Собираем свой компьютер. Логические гейты.
date:   2022-03-27 00:00:00 +0000
categories: jekyll update
---
<small>
На основе курса Nand2Tetris и книги The Elements of Computing Systems
</small>

Чтобы построить свой компьютер, нужно сначала разобраться с тем, как работает “железо”. Если обобщать, то оно состоит из плат и чипов, которые в свою очередь состоят из логических вентилей (logic gate), которые оперируют нулями и единицами - булевой алгеброй. Каждый гейт можно представить как некую математическую функцию, а описать эту функцию можно рисунком, таблицей истинности или языком разметки HDL - Hardware Description Language. В этом посте будут описаны базовые гейты и вариант их реализации на основе Nand гейта:

| a | b | Nand |
|---|---|------|
| 0 | 0 | 1    |
| 0 | 1 | 1    |
| 1 | 0 | 1    |
| 1 | 1 | 0    |


Также я буду использовать те вентили, которые уже описал. 

## Основные гейты

### Not

Просто инвертирует ввод. С 1 на 0 и с 0 на 1.

Схема:

![Not](\assets\2022-03-24-build-computer-logic-gate\not.png)

HDL:

```
CHIP Not {
    IN in;
    OUT out;

    PARTS:
    Nand(a = in, b = in, out = out);
}
```

#### HDL Разметка, основы
    
Для описания чипа через HDL мы сначала пишем ключевое слово CHIP, затем имя разрабатываемого чипа. Затем внутри фигурных скобочек мы описываем две вещи:
    
* Входы\выходы чипа (”пины”)
* Внутренности чипа (чип может состоять из других чипов)
    
В данном случае мы говорим, что у чипа Not снаружи один пин для получения информации: in и один, который отдает данные: out. Сам чип Not состоит из одного чипа Nand.

Внутри круглых скобочек мы описываем, как пины чипа Nand соединены с пинами нашего чипа. А точнее, оба пина a и b соединены с одним и тем же пином in, а пин Nand out соединён с пином out нашего разрабатываемого чипа.
    
### And

Возвращает 1, только если все аргументы 1. Иначе 0.

Схема:

![And](\assets\2022-03-24-build-computer-logic-gate\and.png)

```
CHIP And {
    IN a, b;
    OUT out;

    PARTS:
    Nand(a = a, b = b, out = nandOut);
    Not(in = nandOut, out = out);
}
```

### Or

Возвращает 1, если хотя бы один из аргументов 1.

Схема:

![Or](\assets\2022-03-24-build-computer-logic-gate\or.png)

```
CHIP Or {
    IN a, b;
    OUT out;

    PARTS:
    Not(in = a, out = notA);
    Not(in = b, out = notB);
    Nand(a = notA, b = notB, out = out);
}
```

### Xor

Возвращает 1, если аргументы разные.

Схема:

![Xor](\assets\2022-03-24-build-computer-logic-gate\xor.png)

```
CHIP Xor {
    IN a, b;
    OUT out;

    PARTS:
    Nand(a = a, b = b, out = nandOut);
    Or(a = a, b = b, out = orOut);
    And(a = nandOut, b = orOut, out = out);
}
```

### Mux

Мультиплексор. На вход получает три сигнала. Если третий сигнал 0, то на выход подается первый сигнал. Иначе - второй. 

Схема:

![Mux](\assets\2022-03-24-build-computer-logic-gate\mux.png)

```
CHIP Mux {
    IN a, b, sel;
    OUT out;

    PARTS:
    
    Not(in = sel, out = notSel);
    And(a = a, b = notSel, out = aAnd);
    And(a = sel, b = b, out = bAnd);
    Or(a = aAnd, b = bAnd, out = out);
}
```

### DMux

Демультиплексор. Передает один из вводов на заданный сигнал вывода. Легче всего проиллюстрировать таблицей истинности:

| In | Sel | Out1 | Out2 |
|----|-----|------|------|
| 0  | 0   | In   | 0    |
| 0  | 1   | 0    | In   |
| 1  | 0   | In   | 0    |
| 1  | 1   | 0    | In   |


Или, если подставить значения:


| In | Sel | Out1 | Out2 |
|----|-----|------|------|
| 0  | 0   | 0    | 0    |
| 0  | 1   | 0    | 0    |
| 1  | 0   | 1    | 0    |
| 1  | 1   | 0    | 1    |

Иными словами

```markdown
  IN       OUT
{a, b} = {in, 0} if sel == 0
         {0, in} if sel == 1
```
Схема:

![DMux](\assets\2022-03-24-build-computer-logic-gate\dmux.png)

Реализация:

```
CHIP DMux {
    IN in, sel;
    OUT a, b;

    PARTS:
    Not(in = sel, out = notSel);
    And(a = in, b = notSel, out = a);
    And(a = in, b = sel, out = b);
}
```

### And16
Также существуют гейты с шинами сигналов. Шина сигналов - N сигналов, объединенных в один, можно привести аналогию с массивом. Для примера приведу только And16 - And гейт для шины из 16 сигналов. Остальные гейты реализовываются по аналогии.

```
CHIP And16 {
    IN a[16], b[16];
    OUT out[16];

    PARTS:
    And(a = a[0], b = b[0], out = out[0]);
    And(a = a[1], b = b[1], out = out[1]);
    And(a = a[2], b = b[2], out = out[2]);
    And(a = a[3], b = b[3], out = out[3]);
    And(a = a[4], b = b[4], out = out[4]);
    And(a = a[5], b = b[5], out = out[5]);
    And(a = a[6], b = b[6], out = out[6]);
    And(a = a[7], b = b[7], out = out[7]);
    And(a = a[8], b = b[8], out = out[8]);
    And(a = a[9], b = b[9], out = out[9]);
    And(a = a[10], b = b[10], out = out[10]);
    And(a = a[11], b = b[11], out = out[11]);
    And(a = a[12], b = b[12], out = out[12]);
    And(a = a[13], b = b[13], out = out[13]);
    And(a = a[14], b = b[14], out = out[14]);
    And(a = a[15], b = b[15], out = out[15]);
}
```

В данном примере используются 3 шины сигналов: **a**, **b** и **out**. В каждой шине 16 сигналов. Мы получаем доступ к каждому конкретному через квадратные скобки. 

## Расширенные версии Mux\DMux

Если DMux можно было описать как 

```markdown
  IN       OUT
{a, b} = {in, 0} if sel == 0
         {0, in} if sel == 1
```

То DMux4Way:

```markdown
     IN             OUT
{a, b, c, d} = {in, 0, 0, 0} if sel == 00
               {0, in, 0, 0} if sel == 01
               {0, 0, in, 0} if sel == 10
               {0, 0, 0, in} if sel == 11
```

Аналогично, если Mux

```markdown
out = a if sel == 0
      b if sel == 1
```

То Mux4Way

```markdown
out = a if sel == 00
      b if sel == 01
      c if sel == 10
      d if sel == 11
```

Реализации

### Mux4Way

```
CHIP Mux4Way {
    IN a, b, c, d, sel[2];
    OUT out;

    PARTS:
    Mux(a = a, b = b, sel = sel[0], out = mux0);
    Mux(a = c, b = d, sel = sel[0], out = mux1);

    Mux(a = mux0, b = mux1, sel = sel[1], out = out);
}
```

### DMux4Way

```
CHIP DMux4Way {
    IN in, sel[2];
    OUT a, b, c, d;

    PARTS:
    DMux(in=in, sel=sel[1], a=ab, b=cd);
    
    DMux(in=ab, sel=sel[0], a=a, b=b);
    DMux(in=cd, sel=sel[0], a=c, b=d);
}
```