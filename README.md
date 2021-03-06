## 1. Wprowadzenie do stosu
Czy rozumiesz, czym jest rama stosu?
Śledźmy następujący kod i zobaczmy, co dzieje się na stosie.
```
call function  ; [1]
mov edx, eax
...

function:
push rbp       ; [2]
mov rbp, rsp   ; [3]
sub rsp, 0x10  ; [3]
lea rdi, [rbp-0x10]
call gets
leave          ; [4]
ret            ; [5]
```

Po pierwsze, kod nazywa sie `function` w [1].
Przed wywołaniem funkcji początkowy stos wygląda jak (a) na poniższym rysunku.

Po przebiegu instrukcji `call`, adres zwrotny jest pchany na szczyt stosu jak (b).
Adres powrotny jest adresem następnej instrukcji po rozmówcy.
Tym razem adres `mov edx, eax` jest pchana na stos.
```
|          |       |          |       |          |       |          |
+----------+       +----------+       +----------+       +rsp-------+
|          |       |          |       |          |       |  local   |
|          |       |          |       |          |       | variables|
+----------+       +----------+       +rsp-------+       +rbp-------+
|          |       |          |       | saved bp |       | saved bp |
+----------+  [1]  +rsp-------+  [2]  +----------+  [3]  +----------+
|   ....   | ----> | ret addr | ----> | ret addr | ----> | ret addr |
+rsp-------+       +----------+       +----------+       +----------+
|   ....   |       |   ....   |       |   ....   |       |   ....   |
    (a)                (b)                (c)                (d)
```
Teraz jesteśmy w funkcji callee.
Funkcja tworzy ramkę stosu w [2] i [3].
Po pierwsze, oszczędza wskaźnik podstawowy (RBP) w [2] jak (c) na rysunku.
RBP służy do odwoływania się do zmiennych lokalnych.
Jak wchodzimy do innego zakresu, musimy zapisać wskaźnik podstawowy.

Następnie nowy wskaźnik bazowy jest ustawiony na top stos (RSP) w [3].
Gdy funkcja używa zmiennych/niektórych lokalnych, całkowity rozmiar zmiennych jest odejmowane od RSP jak (D).
Ta sekwencja operacji może być wykonana przez `enter` instrukcji.
Jednak ta instrukcja rzadko jest używana z powodu problemów z wydajnością.

Funkcja może teraz korzystać z zmiennej lokalnej, odwołując się do [rbp-0x ??].
Następnie zobaczmy, co się dzieje przy opuszczeniu funkcji.
```
|          |       |          |       |          |
+rsp-------+       +----------+       +----------+
|  local   |       |          |       |          |
| variables|       |          |       |          |
+rbp-------+       +----------+       +----------+
| saved bp |       |          |       |          |
+----------+  [4]  +rsp-------+  [5]  +----------+
| ret addr | ----> | ret addr | ----> |          |
+----------+       +----------+       +rsp-------+
|   ....   |       |   ....   |       |   ....   |
    (d)                (e)                (f)
```
`leave` to instrukcja działa jako `mov rsp, rbp; pob rpb;`.
Ta instrukcja przywraca stan jak (E) na powyższym rysunku.
Wreszcie, instrukcja `ret` wyskakuje wartość na topie stosu i ustawia wskaźnik instrukcji (RIP) do adresu powrotnego.
W ten sposób możemy zadzwonić do dowolnych funkcji z dowolnego miejsca i powrócić do poprawnych miejsc.

## 2. Wprowadzenie do przepelnienia bufora stosu
Jeśli nastąpi przepełnienie bufora, zapisany wskaźnik bazowy i adres zwrotny zostaną nadpisywane poniżej rysunku poniżej.
```
|          |            |          |
+rsp-------+            +rsp-------+
|  local   |            | AAAAAAAA |
| variables|            | AAAAAAAA |
+rbp-------+            +rbp-------+
| saved bp |   buffer   | AAAAAAAA |
+----------+  overflow  +----------+
| ret addr | ---------> | BBBBBBBB |
+----------+            +----------+
|   ....   |            |   ....   |
```
W takim przypadku RBP staje się "AAAAAAAA" i `ret` instrukcja próbuje przejść do "BBBBBBB".
Pweners nadużywają tę lukę, aby kontrolować RIP do gdziekolwiek chcą skakać.

## 3. Projekt tego wyzwania
W tym wyzwaniu program określa własne instrukcje `call`/`ret`.
```
%macro call 1
;; __stack_shadow[__stack_depth++] = return_address;
  mov ecx, [__stack_depth]
  mov qword [__stack_shadow + rcx * 8], %%return_address
  inc dword [__stack_depth]
;; goto function
  jmp %1
  %%return_address:
%endmacro

%macro ret 0
;; goto __stack_shadow[--__stack_depth];
  dec dword [__stack_depth]
  mov ecx, [__stack_depth]
  jmp qword [__stack_shadow + rcx * 8]
%endmacro
```
Nie zapisuje adresu powrotnego na stosie, ale zapisuje go w tablicy w sekcji BSS.
Ponieważ nie mamy adresu powrotnego na stosie, atakujący nie może nadużywać przepełnienia stosu, aby nadpisać adres zwrotny :)

## 4. Wskazówka.
Więc nie możesz po prostu zastąpić adresu zwrotnego.
Jak wspomniano w rozdziale 2, atakujący może zastąpić nie tylko adres zwrotny, ale także zapisany wskaźnik podstawy.
Co może zrobić atakujący, nadpisując wskaźnik podstawowy?

Nie zapomnij również sprawdzić mechanizmu bezpieczeństwa programu (to znaczy SSP, DEP i PIE).

----

Mam nadzieję, że teraz zrozumiesz, jak tworzona jest rama stosu, a także jak działa `call` i `ret`!
