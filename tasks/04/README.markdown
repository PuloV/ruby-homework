# Inline assembler

Езиците от по-ниско ниво позволяват вграждането на Assembler. Например в Pascal
можете да имате следния фрагмент:

    begin
        width := 320;   {код на Pascal}
        height := 200;  {още код на Pascal}

        asm
            mov ax,13h  ; x86 assembler
            int 10h     ; x86 assembler
        end;

        ...
    end;

Тъй като Ruby е език от много високо ниво самата идея да добавим подобна
функционалност е в разрез с идеите, стоящи зад него. Затова пък ще можем да
упражним създаване на DSL-и. И така:

Ще симулираме много малко подмножество на
[x86 архитектурата](http://en.wikipedia.org/wiki/X86_assembly).
Цялата памет, с която разполагаме ще бъде 4 регистъра, в които ще могат да се
пазят целочислени стойности. Регистрите се казват `ax`, `bx`, `cx` и `dx`.
Преди изпълнението на програмата всеки от тях има стойност `0`. Освен
регистрите, разпогаме със следните инструкции:

1. `mov` <em>destination\_register</em>, <em>source</em>

   `mov` има два аргумента. <em>source</em> е число или име на регистър, от
   който да бъде взета числовата стойност, а <em>destination\_register</em> е
   името на регистъра, в който ще бъде записана числовата стойност. Примери:

   * `mov ax, 2` - записва в регистъра `ax` стойността `2`, нещо като `ax = 2`.
   * `mov bx, dx` - записва в регистъра `bx` стойността, съдържаща се в регистъра
     `dx`, нещо като `bx = dx`.

1. `inc` <em>destination_register</em>, <em>value</em>

   `inc` прибавя <em>value</em> към стойността, съдържаща се в
   <em>destination\_register</em> и запазва получената стойност в
   <em>destination\_register</em>. <em>value</em> може да бъде число,
   име на регистър, от който да бъде взета числовата стойност, а може и
   да не се подава като аргумент, в който случай се използва стойността
   по-подразбиране – `1`. Примери:

   * `inc ax, 3` - увеличава стойността, записана в `ax` с 3, нещо като `ax += 3`.
   * `inc bx, dx` - увеличава стойността, записана в `bx` със стойността,
     записана в `dx`, нещо като `bx += dx`.
   * `inc cx` - увеличава стойността, записана в `cx` с единица, нещо като `cx += 1`.

1. `dec` <em>destination_register</em>, <em>value</em>

   `dec` изважда <em>value</em> от стойността, съдържаща се в
   <em>destination\_register</em> и запазва получената стойност в
   <em>destination\_register</em>. <em>value</em> може да бъде число, име на
   регистър, от който да бъде взета числовата стойност, а може и да не се
   подава като аргумент, в който случай се използва стойността по-подразбиране
   – `1`. Примери:

   * `dec ax, 3` - намалява стойността, записана в `ax` с 3, нещо като `ax -= 3`.
   * `dec bx, dx` - намалява стойността, записана в `bx` със стойността,
     записана в `dx`, нещо като `bx -= dx`.
   * `dec cx` - намалява стойността, записана в `cx` с единица, нещо като `cx -= 1`.
1. `cmp` <em>register</em>, <em>value</em>

   `cmp` сравнява стойността, съдържаща се в <em>register</em> с <em>value</em>,
    където <em>value</em> може да бъде число, или име на регистър, от който да бъде
    взета числовата стойност. Резултатът от сравнението трябва да бъде запомнен
    някъде, вие избирате къде, за да може да бъде използван по-нататък. Резултатът
    се определя по следния начин:

    * нула, ако <em>register</em> == <em>value</em>
    * отрицателно число, ако <em>register</em> < <em>value</em>
    * положително число, ако <em>register</em> > <em>value</em>

    Прилича ви на `<=>`, нали? Пример:

        mov ax, 3
        mov bx, 2

        cmp ax, 3        ; тук резултатът е 0
        cmp ax, 10       ; тук резултатът е отрицателен
        cmp ax, bx       ; тук резултатът е положителен

1. `label` <em>label_name</em>

   `label` служи, за да постави маркер в програмата, към който да можем да се върнем
   по-късно с някакъв вид **jump**.

1. `jmp` <em>where</em>

   Безусловен преход към инструкция. <em>where</em> може да бъде име на етикет,
   зададен с `label`, или число – пореден номер на инструкция. Номерацията на
   инструкциите е от `0`, като `label` **не се брои за инструкция**. Примери:

        mov ax, 3        ; инструкция 0
        inc ax           ; инструкция 1
        dex ax           ; инструкция 2
        jmp 1            ; след този jmp изпълнението преминава на инструкция 1

        mov ax, 3        ; инструкция 0
        mov bx, 3        ; инструкция 1
        label jump_here
        inc ax           ; инструкция 2
        dex ax           ; инструкция 3
        jmp jump_here    ; след този jmp изпълнението преминава на инструкция 2

1. `je` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` е нула. Ако резултатът от
   последния `cmp` не е нула, просто се преминава на следващата инструкция. Пример:

        mov ax, 2
        cmp ax, 2
        je finish
        mov bx, 3        ; тази инструкция никога не се изпълнява
        label finish

        mov ax, 2
        cmp ax, 3
        je finish
        mov bx, 3        ; тази инструкция ще се изпълни, понеже 2 не е равно на 3
        label finish

1. `jne` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` не е нула. Ако
   резултатът от последния `cmp` е нула, просто се преминава на следващата
   инструкция.

1. `jl` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` е отрицателен. Ако
   резултатът от последния `cmp` е нула, или е положителен, просто се преминава
   на следващата инструкция.

1. `jle` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` е отрицателен или е
   нула. Ако резултатът от последния `cmp` е положителен, просто се преминава
   на следващата инструкция.

1. `jg` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` е положителен. Ако
   резултатът от последния `cmp` е нула, или е отрицателен, просто се преминава
   на следващата инструкция.

1. `jge` <em>where</em>

   Преход към инструкция, както при `jmp`, но този път преходът трябва да се
   случи само ако резултат от последно изпълненият `cmp` е положителен или е
   нула. Ако резултатът от последния `cmp` е отрицателен, просто се преминава
   на следващата инструкция.

Създайте модул `Asm` съдържащ метод `Asm.asm`, който приема блок,
съдържащ последователност от инструкции, които ще бъдат описани по-надолу, и
връща масив, съдържащ стойностите на регистрите `ax`, `bx`, `cx` и `dx`.

Пример за завършена програма, която намира най-големия общ делител на две
числа (a.k.a. „Алгоритъм на Евклид“):

    Asm.asm do
      mov ax, 40
      mov bx, 32
      label cycle
      cmp ax, bx
      je finish
      jl asmaller
      dec ax, bx
      jmp cycle
      label asmaller
      dec bx, ax
      jmp cycle
      label finish
    end                 # => [8, 8, 0, 0]
