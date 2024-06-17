# 修改ELF裡面的instruction

目標: 手寫一段程式，取代一個ELF中的某個function

## 步驟1 - 手寫一段程式，把它編譯好並且轉成binary
```
vi my_func_new.c
riscv64-unknown-linux-gnu-gcc -c my_func_new.c -o my_func_new.o
riscv64-unknown-linux-gnu-objcopy -O binary my_func_new.o my_func_new.bin
```

## 步驟2 - 檢查要取代的function大小夠不夠大
1. 先看看symbol table dump出來的格式:

```
Symbol table '.dynsym' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000610     0 SECTION LOCAL  DEFAULT   12 .text
     ...
     8: 00000000000006fe    48 FUNC    GLOBAL DEFAULT   12 main

Symbol table '.symtab' contains 71 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000270     0 SECTION LOCAL  DEFAULT    1 .interp
     2: 0000000000000294     0 SECTION LOCAL  DEFAULT    2 .note.ABI-tag
     3: 00000000000002b8     0 SECTION LOCAL  DEFAULT    3 .hash
    ...
    57: 00000000000006c8    32 FUNC    GLOBAL DEFAULT   12 hello
    ...
```
2. 從上面的格式中撈出要改的function size
```
elf=hello_world.elf
symbol=hello
riscv64-unknown-linux-gnu-readelf -s $elf \
| sort -k2 \
| awk -v symbol=$symbol 'BEGIN{show = 0}; {
    if ($0 ~ "\\<" symbol "\\>" ) { show = 1 }
    if (show == 1 && $2 ~ /\<[0-9a-f]+\>/ ) { print strtonum("0x" $2) | "uniq" }}' \
| awk 'NR==1{a=$0}; NR==2{printf "0x%x %d\n", a, $0-a}'
```

執行結果

```
0x6c8 32
```
第一欄是該function的起始PC, 第二欄是size. 跟步驟1的binary size比較，如果binary size比較大，那蓋上去ELF時就會蓋到別的function，需要重新設計要修改的程式

## 步驟3 - 找出該function位於ELF的哪裡
用 `objdump -F` 觀察function的位置
```
$ riscv64-unknown-linux-gnu-objdump -Fd hello_world.elf
...
00000000000006c8 <hello> (File Offset: 0x6c8):
 6c8:   1141                    add     sp,sp,-16
 6ca:   e406                    sd      ra,8(sp)
 6cc:   e022                    sd      s0,0(sp)
 6ce:   0800                    add     s0,sp,16
 6d0:   00000517                auipc   a0,0x0
 6d4:   06850513                add     a0,a0,104 # 738 <_IO_stdin_used+0x8> (File Offset: 0x738)
 6d8:   f19ff0ef                jal     5f0 <puts@plt> (File Offset: 0x5f0)
 6dc:   0001                    nop
 6de:   853e                    mv      a0,a5
 6e0:   60a2                    ld      ra,8(sp)
 6e2:   6402                    ld      s0,0(sp)
 6e4:   0141                    add     sp,sp,16
 6e6:   8082                    ret
...
```
小工具
```
elf=hello_world.elf
symbol=hello
riscv64-unknown-linux-gnu-objdump -Fd $elf | grep "^[0-9a-f]\+ <$symbol>" | awk '{print $5}' | sed 's/)://'
```
執行結果
```
0x6c8
```

## 步驟4 - 把新的binary蓋到ELF上
```
dd bs=1 if=my_func_new.bin of=hello_world.elf seek=$((0x6c8)) conv=notrunc
```
