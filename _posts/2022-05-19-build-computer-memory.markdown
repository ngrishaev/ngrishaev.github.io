---
layout: post
title: Собираем свой компьютер. Память
date:   2022-05-19 00:00:00 +0000
categories: jekyll update
---
<small>
На основе курса Nand2Tetris и книги The Elements of Computing Systems
</small>

Концепция памяти невозможна без концепции времени. Мы помним сейчас, что произошло раньше. Поток времени в компьютере обычно представлен в виде часов, которые постоянно отправляют поток изменяющихся сигналов. Обычно это сигнал, меняющийся между двумя значениями 0 - 1 \ low - high \ false - true и так далее. Для дальнейшей разработки компьютера нам нужен еще один элементарный элемент: Flip-Flop (Триггер), сокращенно FF. Это общее название целой группы чипов, в этих постах я буду использовать DFF.

## DFF

DFF - Data (Delay) Flip Flop имеет однобитовый вход и выход. На выход он подает то, что получил в прошлый тик времени на вход. Можно сказать, что он работает так: 

```
out(t) = in(t - 1) 
```

Чтобы следить за тиками времени, у него есть еще один скрытый вход, которым он соединен с главными часами в компьютере. В самый первый тик (t = 0) из DFF выходит сигнал 0. Я буду использовать DFF как элементарный чип, то есть без раскрытия его реализации.

## Регистр

Чип, который хранит данные. У него есть два входа: нагрузка и данные. Если в нагрузку поступает сигнал, то чип запоминает данные на входе.

```
if (load == 1) out(t + 1) = in(t)
if (load == 0) out(t + 1) = out(t)
```

Регистры бывают разной длины. Они могут хранить как один бит, так и несколько. Количество битов, хранящихся в регистре, обычно и принимают за длину слова (подробнее в [прошлом посте]({% post_url 2022-05-09-build-computer-alu %})).

### Реализация регистра с единственной ячейкой памяти (бит):

```
CHIP Bit {
    IN in, load;
    OUT out;

    PARTS:

    Mux(a = dffOut, b = in, sel = load, out = muxOut);
    DFF(in = muxOut, out = dffOut, out = out);
}
```
[![Bit](/assets/2022-05-19-build-computer-memory/Bit.png)](/assets/2022-05-19-build-computer-memory/Bit.png)

### Регистр с несколькими битами (4):

```
CHIP Register {
    IN in[4], load;
    OUT out[4];

    PARTS:
    Bit(in = in[0], load = load, out = out[0]);
    Bit(in = in[1], load = load, out = out[1]);
    Bit(in = in[2], load = load, out = out[2]);
    Bit(in = in[3], load = load, out = out[3]);
}
```

[![Register](/assets/2022-05-19-build-computer-memory/Register.png)](/assets/2022-05-19-build-computer-memory/Register.png)

## RAM

Собственно, память. Состоит из множества регистров (четырех, в данной реализации). Имеет три входа: нагрузка, данные и адрес. Первые два - то же самое, что и у регистров. Адрес - в какой именно регистр мы хотим записать данные или в какой именно регистр хотим посмотреть.

### Реализация

В чипе DMux мы выбираем, какой именно регистр мы будем использовать. В блоке чипов And мы для каждого используемого чипа выбираем, что у него будет подано на вход  load.

Таким образом, мы обращаемся сразу ко всем чипам, но load передаем только по нужному адресу. В Mux чипе мы делаем обратную DMux операцию - выбираем по адресу, данные из какого регистра будут на выходе.

```
CHIP RAM4 {
    IN in[4], load, address[2];
    OUT out[4];

    PARTS:

    DMux4Way(
        in = true,
        sel = address,
        a = r1Sel,
        b = r2Sel,
        c = r3Sel,
        d = r4Sel);

    And(a = r1Sel, b = load, out = r1Load);
    And(a = r2Sel, b = load, out = r2Load);
    And(a = r3Sel, b = load, out = r3Load);
    And(a = r4Sel, b = load, out = r4Load);

    Register(in = in, load = r1Load, out = r1Out);
    Register(in = in, load = r2Load, out = r2Out);
    Register(in = in, load = r3Load, out = r3Out);
    Register(in = in, load = r4Load, out = r4Out);

    Mux4Way(
        a = r1Out,
        b = r2Out,
        c = r3Out,
        d = r4Out,
        sel = address,
        out = out
    );
}
```

[![RAM](/assets/2022-05-19-build-computer-memory/RAM.png)](/assets/2022-05-19-build-computer-memory/RAM.png)

## Program Counter

Этот чип нам пригодится, когда будем реализовывать ассемблер и работу с байт кодом.

Имеет четыре входа: in - вход с данным, и 3 управляющих флага: load, inc, reset, и один выход.

- Если никакой флаг не был задействован, то состояние счетчика не меняется.
- Если был сигнал в inc флаге, то значение в счетчике увеличивается на 1.
- Если reset, то значение обнуляется.
- Если load, то в счетчик записывается значение, которое было на in входе.

В группе Mux чипов мы считаем, что должно быть записано в счетчик (если должно).

В группе Or чипов мы считаем, нужно ли полученное в Mux чипах записывать.

```
// if      (reset[t] == 1) out[t+1] = 0
// else if (load[t] == 1)  out[t+1] = in[t]
// else if (inc[t] == 1)   out[t+1] = out[t] + 1  (integer addition)
// else                    out[t+1] = out[t]

CHIP PC {
    IN in[4],load,inc,reset;
    OUT out[4];

    PARTS:

    Inc4(in = registerOut, out = regInc);

    Mux4(
        a = false,
        b = regInc,
        sel = inc,
        out = muxInc);
    Mux4(
        a = muxInc,
        b = in,
        sel = load,
        out = muxLoad);
    Mux4(
        a = muxLoad,
        b = false,
        sel = reset,
        out = muxReset);

    Or(a = inc, b = load, out = incOrLoad);
    Or(a = incOrLoad, b = reset, out = incOrLoadOrReset);

    Register(
        in = muxReset,
        load = incOrLoadOrReset,
        out = registerOut,
        out = out);        
}
```

[![RAM](/assets/2022-05-19-build-computer-memory/PC.png)](/assets/2022-05-19-build-computer-memory/PC.png)