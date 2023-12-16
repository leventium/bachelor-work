# DarkRISCV

~~Реализован с нуля за одну ночь~~

- [GitHub](https://github.com/darklife/darkriscv)
- Отдельной документации нет


## Обзор

DarkRISCV - Софт-процессор написанный на Verilog реализующее RV32I и RV32E, в
относительности небольших размеров описание позволяет с меньшими трудностями
разобраться и модифицировать. FPGA Proven, загружался на ПЛИС от xilinx и
некоторые от Altera (но, к сожалению, не на наши).

Некоторые возможности ядра:

- Нормально работает с GCC
- Поддержка UART
- Контроллер прерываний
- Контроллер кэшэй
- GPIO и таймер
- Предсказатель ветвления