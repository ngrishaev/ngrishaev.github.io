---
layout: post
title: Собираем свой компьютер. Компьютер
date:   2022-06-05 00:00:00 +0000
categories: jekyll update
---
<small>
На основе курса Nand2Tetris и книги The Elements of Computing Systems
</small>

На этом посте заканчивается “железная” часть компьютера, дальше остается только софт - интерпретаторы, ассемблеры, виртуальная машина и прочее. Сначала разберемся с машинным кодом и принципиальным устройством компьютера.

## Компьютер

Общая принципиальная схема компьютера:
[![Computer](/assets/2022-06-05-build-computer-computer/computer_general.png)](/assets/2022-06-05-build-computer-computer/computer_general.png)

В нашем случае CPU (процессор) - это, грубо говоря, ALU, созданный ранее.  Память состоит из нескольких RAM чипов (для данных) и ROM (read only memory) памяти, где хранится программа. Обычно программа не является частью компьютера. Как пример - картриджи для NES. Их обычно так и называют: “Contra ROM for NES”, “Mario ROM for NES”.

В программе в пронумерованном порядке лежат машинные коды для CPU. Машинный код - формализованная запись команды для CPU. По сути - набор всех инпутов для ALU, упорядоченный особым образом.

### Иерархия памяти

Чем больший размер памяти мы используем, тем дороже нам обходится чтение из нее. Это связано с тем, что в больших блоках памяти придется использовать адрес длиннее, чем в коротких. Чтобы смягчить эту проблему, мы используем иерархию памяти: у нас есть совсем маленькие участки памяти (вплоть до регистров), которые обращаются к участкам побольше, которые обращаются к участкам побольше и так далее. Самые высокие уровни памяти обычно расположены прямо в процессоре. Схема примерно такая: регистры процессора → RAM → жесткий диск. Обычно регистры являются частью машинных кодов. 

В нашей реализации у CPU будет 3 регистра: D, A, M:

- D - Data register. В нем просто можно хранить данные.
- A - Address register. Также можно использовать для хранения данных. Кроме этого, является ссылкой на значение в регистре M
- M - Memory register. Ссылается на участок памяти RAM[A]. Через этот регистр процессор будет взаимодействовать с памятью. Не является регистром в физическом смысле.

### Ввод/вывод

В качестве ввода/вывода будут использоваться клавиатура и черно-белый монитор. По сути они оба являются участками памяти.

Каждому пикселю монитора ставится в соответствие определенный бит в выделенном промежутке памяти. Если этот бит установлен в 1, то на мониторе этот пиксель загорается черным. Иначе он остается белым.

Клавиатура - наоборот: пишет в выделенный ей участок памяти какое-либо значение, которому соответствует нажатая клавиша. Если на клавиатуре никто не нажимает кнопки в данный момент, то в регистре будет просто 0.

### Program Counter (PC)

Еще один регистр, расположенный в процессоре. Используется для управления потоком выполнения программы. В нем хранится адрес команды, которая будет выполнена следующей. При выполнении машинной команды автоматически инкрементируется, что позволяет в следующий тик выполнить следующую команду. Кроме того, существуют специальные jump команды, которые позволяют установить PC в нужный адрес. С их помощью реализуются всевозможные циклы и if операторы. 

### A/C Команды

Подробнее спецификацию кодов можно посмотреть в книге The Elements of Computing, которая частями выложена на сайте [nand2tetris](https://www.nand2tetris.org/_files/ugd/44046b_7ef1c00a714c46768f08c459a6cab45a.pdf).

Будет два вида машинных команд:

- Address command
- Computation command

В прошлых постах чипы были реализованы для 4-битного компьютера, потому что так легче описывать схемы. Для 8\16\32\64 устройство принципиально не отличалось бы.

Но дальше я буду описывать устройство для 16-битного компьютера, так что и регистры у нас будут 16-битные. И, соответственно, команды.

#### A Command

В общем виде A команда выглядит так:

$$0vvv\ vvvv\ vvvv\ vvvv$$

Если команда начинается c “0”, то это А команда. Остальные 15 битов описывают адрес. Адрес не может быть отрицательным числом. Единственное, что делает А команда - записывает адрес в регистр А.

#### C Command

В общем виде C команда выглядит так:

$$111a\ c_1c_2c_3c_4\ c_5c_6d_1d_2\ d_3j_1j_2j_3$$

C команда всегда начинается с 1.

В группе битов **a** и **c** определяется, что именно мы хотим вычислить, какую операцию и над какими регистрами хотим выполнить.

Группа битов **d** определяет, в какой регистр нужно записать результат вычисления.

Группа битов **j** отвечает за управление потоком выполнения команды. Если результат вычисления **c** удовлетворяет условию **j**, то в **PC** запишется адрес, указанный в регистре **А**

## Реализация

### Память

```
CHIP Memory {
    IN in[16], load, address[15];
    OUT out[16];

    PARTS:

    DMux4Way(
        in = load,
        sel = address[13..14],
        a = ramLoad1,
        b = ramLoad2,
        c = screenLoad,
        d = keyboard
        );

    Or(
        a = ramLoad1,
        b = ramLoad2,
        out = ramLoad
        );

    RAM16K(
        in = in,
        load = ramLoad,
        address = address[0..13],
        out = ramOut
        );

    Screen(
        in = in,
        load = screenLoad,
        address = address[0..12],
        out = screenOut
        );

    Keyboard(out = keyboardVal);
    
    // Make sure address <= 0110 0000 0000 0000
    Or8Way(in = address[0..7], out = isFirstHalfZero);
    Or8Way(in[0..4] = address[8..12], in[5..7] = false, out = isSecondHalfZero); 
    Or(a = isFirstHalfZero, b = isSecondHalfZero, out = isAllZeroes);
    Not(in = isAllZeroes, out = isInAddress);

    Mux16(
        a = false,
        b = keyboardVal,
        sel = isInAddress,
        out = keyboardOut
    );

    Mux4Way16(
        a = ramOut,
        b = ramOut,
        c = screenOut,
        d = keyboardOut,
        sel = address[13..14],
        out = out
    );
}
```

Стоит обратить внимание на код после комментария “Make sure address <= 0110 0000 0000 0000”. За этим адресом находится память, не используемая ни для данных, ни для клавиатуры, ни для экрана. Поэтому нам нужно убедиться, что мы не будем пытаться писать в эту область памяти.

### CPU

```
CHIP CPU {

    IN  inM[16],         // M value input  (M = contents of RAM[A])
        instruction[16], // Instruction for execution
        reset;           // Signals whether to re-start the current
                         // program (reset==1) or continue executing
                         // the current program (reset==0).

    OUT outM[16],        // M value output
        writeM,          // Write to M? 
        addressM[15],    // Address in data memory (of M)
        pc[15];          // address of next instruction

    PARTS:
    
    Not(in = instruction[15], out = isACommand);
    Or(a = isACommand, b = instruction[5], out = aLoad);
    And(a = instruction[15], b = instruction[4], out = dLoad);
    And(a = instruction[15], b = instruction[3], out = writeM);

    //A Register Mux
    Mux16(
    a = aluToA,
    b = instruction,
    sel = isACommand,
    out = aMux);

    //A Register
    ARegister(
        in = aMux,
        load = aLoad,
        out = aRegister,
        out[0..14] = addressM
        );

    //ALU Mux
    Mux16(
        a = aRegister,
        b = inM,
        sel = instruction[12],
        out = aluMux); 
    
    //D Register
    DRegister(
        in = aluToD,
        load = dLoad,
        out = dRegister); 

    ALU(
        x = dRegister, 
        y = aluMux,   
        zx = instruction[11],
        nx = instruction[10],
        zy = instruction[9],
        ny = instruction[8],
        f  = instruction[7],
        no = instruction[6],
        out = outM, 
        out = aluToD,
        out = aluToA,
        zr = aluZero, 
        ng = aluNegative 
    );

    Not(in = aluZero, out = notZr);
    Not(in = aluNegative, out = notNr);

    And(a = true, b = instruction[0], out = j3);
    And(a = true, b = instruction[1], out = j2);
    And(a = true, b = instruction[2], out = j1);

    And(a = instruction[2], b = instruction[1], out = j1j2);
    And(a = j1j2, b = aluNegative, out = j1j2Nr);

    And(a = instruction[2], b = notZr, out = j1NZr);
    And(a = j1NZr, b = aluNegative, out = j1NZrNr);

    And(a = instruction[1], b = aluZero, out = j2Zr);
    And(a = j2Zr, b = notNr, out = j2ZrNNr);

    And(a = instruction[0], b = notZr, out = j3NZr);
    And(a = j3NZr, b = notNr, out = j3NZrNNr);

    Or(a = j1j2Nr, b = j1NZrNr, out = firstHalf);
    Or(a = j2ZrNNr, b = j3NZrNNr, out = secondHalf);

    Or(a = firstHalf, b = secondHalf, out = jump);
    And(a = jump, b = instruction[15], out = pcLoad);

    PC(
        in = aRegister,
        load = pcLoad,
        inc = true, 
        reset = reset,
        out[0..14] = pc
    );
}

//          a  c1 c2 c3 c4 c5 c6 d1 d2 d3 j1 j2 j3
// 1  1  1  0  0  0  1  1  0  0  0  0  1  0  0  0
// 15 14 13 12 11 10 9  8  7  6  5  4  3  2  1  0
```
Снизу листинга для удобства расписано, какой бит в шине данных соответствует какому биту в C команде.

### Компьютер

```
CHIP Computer {

    IN reset;

    PARTS:

    ROM32K(
        address = pc,
        out = romOut
    );

    CPU(
        inM = ram,
        instruction = romOut,
        reset = reset,
        pc = pc,
        addressM = addressM,
        writeM = writeM,
        outM = outM);

    Memory(
        in = outM,
        load = writeM,
        address = addressM,
        out = ram
    );
}
```

Просто собираем из уже известных нам чипов компьютер. Вход `reset` означает, хотим ли мы сбросить выполнение программы на начало. Если на этом входе мы считываем сигнал, то устанавливаем Program Counter в 0.