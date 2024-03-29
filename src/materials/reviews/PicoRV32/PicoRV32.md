# PicoRV32

Это ядро мы уже кратко обозревали в одном из предыдущих отчётов, здесь
постараемся сделать более подробно.


## Описание

PicoRV32 - ядро разработанное YosysHQ (те же ребята, что делают парсер
verilog'а для OpenLane), может реализовывать архитектуры RV32E, RV32I, RV32IC,
RV32IM или RV32IMC.

Основные особенности:

- Компактное (750-2000 LUT на Xilinx)[^lut]
- Высокое значение максимальной частоты (250-400 МГц)
- Опциональное IRQ[^irq]


[^lut]: ~6000 на Cyclone  
[^irq]: Не соответствует RISC-V ISA


## Некоторые особенности

В процессе изучения ядра было выявлены следующие особенности:

- Всё ядро реализованно в одном Verilog файле (что, возможно, не очень хорошо)
- Вся документация в readme
- Хорошо задокументированы параметры модулей, собственная реализация IRQ
- Остальное задокументировано плохо (вводы/выводы модулей, внутренее
  устройство)
