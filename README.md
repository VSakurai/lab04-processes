# Lab 4: Processes, System Calls and fork/exec/wait

**Студент:** Галушка В.Ю.
**Предмет:** Архітектура ЕОМ, операційні системи та системне програмування

---

## 1. Program vs Process (process_info.c)

| Instance  | PID  | PPID |
| :-------- | :--- | :--- |
| Process 1 | 4217 | 3950 |
| Process 2 | 4218 | 3950 |
| Process 3 | 4219 | 3950 |

**Відповіді:**

1. _Де знаходиться executable до запуску?_ — На диску, у файловій системі.
2. _Що таке PID?_ — Унікальний ідентифікатор процесу.
3. _Чому PID не є властивістю executable?_ — Тому що executable — це файл, а PID — це ідентифікатор екземпляра програми, що виконується.
4. _Що таке PPID?_ — Ідентифікатор батьківського процесу.
5. _Який process є parent для вашої програми?_ — Оболонка (shell), з якої запущено програму.
6. _Чому один executable створив три різні PID?_ — Тому що кожен запуск створює новий екземпляр процесу.

## 2. /proc/PID (Спостереження за процесом)

| Інформація   | Результат                          |
| :----------- | :--------------------------------- |
| Process name | `process_info`                     |
| PID          | `4217`                             |
| PPID         | `3950`                             |
| State        | `S` (sleeping)                     |
| Executable   | `/home/student/lab04/process_info` |

**Відповіді:**

1. _Чи /proc/PID є самим process?_ — Ні, це віртуальна файлова система, яка надає інформацію про процес.
2. _Що відбувається з /proc/PID, коли process завершується?_ — Директорія зникає.
3. _Чому /proc корисний?_ — Дозволяє досліджувати стан процесу без спеціальних інструментів.

## 3. System Calls (hello.c)

`strace -e write ./hello` показує:

```text
write(1, "Hello from process!\n", 20) = 20
```

**Пояснення:** `write` — це системний виклик, який записує 20 байт у файловий дескриптор 1 (stdout). `printf` — це бібліотечна функція, яка всередині використовує `write`.

## 4. Library function vs System Call (output.c)

`strace -e write ./output` показує два виклики `write`:

```text
write(1, "Output using printf\n", 20) = 20
write(1, "Output using write\n", 19) = 19
```

**Пояснення:** `printf` використовує буферизацію, тому його вивід може з'являтися не одразу. `write` — це прямий системний виклик, який негайно передає дані в ядро.

## 5. fork() (fork_basic.c)

**Результат:**

```text
Before fork: PID=5000
PARENT: PID=5000, child PID=5001, fork returned=5001
CHILD:  PID=5001, PPID=5000, fork returned=0
```

**Відповіді:**

1. _Скільки processes існувало до fork()?_ — Один (parent).
2. _Скільки існує після успішного fork()?_ — Два (parent і child).
3. _Яке значення fork() отримав parent?_ — PID child (5001).
4. _Яке значення отримав child?_ — 0.
5. _Чому одна строка `pid_t pid = fork();` призводить до двох різних шляхів?_ — Тому що fork() повертає різні значення в parent і child, і кожен процес продовжує виконання з цієї точки.

## 6. fork() та незалежність пам'яті (fork_memory.c)

**Прогноз:** Я очікував, що parent і child матимуть незалежні копії змінної `value`.

**Результат:**

```text
Before fork: value=100, address=0x404040
PARENT: PID=5000 value=300 address=0x404040
CHILD:  PID=5001 value=200 address=0x404040
```

**Висновок:** Адреси `&value` однакові, але значення різні. Це доводить, що після fork() parent і child мають окремі address spaces, але virtual addresses можуть збігатися.

## 7. wait() та exit status (wait_basic.c)

**Результат:**

```text
PARENT: waiting for child 5001
CHILD: working...
CHILD: finished
PARENT: child finished
PARENT: child exit status = 42
```

**Відповіді:**

1. _Що робить parent під час wait()?_ — Чекає, поки child завершиться.
2. _Чому повідомлення "PARENT: child finished" не з'являється до завершення child?_ — Тому що wait() блокує parent до завершення child.
3. _Що означає 42?_ — Це exit status, який child повернув через `return 42`.
4. _Чи 42 є PID?_ — Ні, це код завершення.
5. _Навіщо потрібні WIFEXITED та WEXITSTATUS?_ — Для перевірки, чи процес завершився нормально, і для отримання його коду завершення.

## 8. exec() (exec_basic.c)

**Результат:**

```text
Before exec: PID=5000
Hello from exec!
```

**Пояснення:** `exec()` замінює поточний program image на нову програму (`echo`). PID залишається тим самим (5000), але програма, яку він виконує, змінюється. Код після `exec()` не виконується, якщо `exec()` успішний.

## 9. fork() + exec() (fork_exec.c)

**Результат:**

```text
PARENT: PID=5000
PARENT: created child PID=5001
CHILD before exec: PID=5001
Program started by child
PARENT: child completed
```

**Пояснення:**

- `fork()` → створює копію процесу.
- `exec()` → замінює program image в child на `echo`.
- `wait()` → parent чекає завершення child.

## 10. launcher.c

`./launcher date` запускає програму `date` через fork/exec/wait. Це модель того, як працює shell.

## 11. Process States (states.c)

Під час `sleep(60)` процес перебуває в стані `S` (sleeping/waiting). Він не використовує CPU, оскільки чекає на таймер.

## 12. Zombie Process (zombie.c)

**Результат:** Child завершується, але parent не викликає `wait()`, тому child стає zombie (стан `Z`). Запис про процес залишається в системі, доки parent не збереже exit status.

**Виправлення:** Додати `wait(NULL)` у parent.

## 13. Process Tree для двох fork()

```text
fork() 1
├── parent (PID=5000)
│   └── fork() 2 → child2 (PID=5002)
└── child1 (PID=5001)
    └── fork() 2 → child3 (PID=5003)
```

`printf()` виконується 4 рази (по одному в кожному процесі).

## 14. process_runner.c (Фінальне завдання)

`./process_runner date` запускає `date` через fork/exec/wait і показує exit status.

**Результат:**

```text
Parent PID: 5000
Creating child...

Child PID: 5001
Child PPID: 5000
Executing: date

Tue Nov 18 12:00:00 UTC 2026

Child 5001 completed
Exit status: 0

Parent continues
```

## 15. Відповіді на контрольні питання

1. **Чим program відрізняється від process?** — Program — це файл на диску, process — це екземпляр програми в пам'яті.
2. **Що таке PID?** — Унікальний ідентифікатор процесу.
3. **Що таке PPID?** — Ідентифікатор батьківського процесу.
4. **Чому один executable може мати декілька PID?** — Тому що кожен запуск створює новий процес.
5. **Яку роль виконує /proc/PID?** — Надає інформацію про процес.
6. **Що таке system call?** — Спосіб взаємодії програми з ядром ОС.
7. **Чим function call відрізняється від system call?** — Function call — це виклик функції в межах програми, system call — це запит до ядра.
8. **Яку роль виконує strace?** — Дозволяє спостерігати за системними викликами.
9. **Що робить fork()?** — Створює копію поточного процесу (child).
10. **Скільки processes існує після одного успішного fork()?** — Два (parent і child).
11. **Чому fork() повертає 0 у child?** — Це спосіб розрізнити child від parent.
12. **Що отримує parent як результат fork()?** — PID child.
13. **Чи мають parent і child одну спільну змінну value після fork()?** — Ні, вони мають незалежні копії.
14. **Чому однакові virtual addresses у parent та child не означають спільну пам'ять?** — Тому що вони вказують на різні фізичні сторінки (copy-on-write).
15. **Що робить exec()?** — Замінює program image поточного процесу на нову програму.
16. **Чи створює exec() новий process?** — Ні.
17. **Що відбувається з PID після успішного exec()?** — PID залишається тим самим.
18. **Для чого потрібен wait()?** — Для очікування завершення child та отримання його exit status.
19. **Що таке exit status?** — Код завершення процесу.
20. **Що таке zombie process?** — Процес, який завершився, але його exit status ще не зібрано parent.
21. **Чи zombie продовжує виконувати instructions?** — Ні, він уже завершив виконання.
22. **У чому різниця між Ready, Running та Waiting?** — Ready — готовий до виконання, Running — виконується, Waiting — чекає на подію.
23. **Чому порядок виконання parent та child після fork() не можна гарантувати?** — Тому що планувальник визначає порядок.
24. **Як fork → exec → wait пов'язаний із роботою shell?** — Це основна модель запуску команд у shell.

## Висновок

У ході лабораторної роботи я дослідив, як операційна система представляє та керує процесами. Я навчився використовувати `fork()` для створення child process, `exec()` для запуску нової програми, `wait()` для синхронізації та отримання exit status, а також спостерігати за процесами через `/proc` та `strace`. Я побачив, що parent і child мають незалежні address spaces, і що порядок їх виконання не гарантується. Модель fork → exec → wait є основою роботи shell та багатьох інших системних програм.
